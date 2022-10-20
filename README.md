### Pro EasyWeChat 5 对接小程序支付(在pro_s_V2.2.2中)
1. 小程序支付管理服务介绍[官方文档](https://developers.weixin.qq.com/miniprogram/dev/platform-capabilities/business-capabilities/ministore/wxafunds/Introduction.html)
   1. 介绍
      >为了优化消费者的支付体验，微信小程序提供了小程序支付管理服务。你可以在小程序后台或通过接口申请微信支付商户号。成功申请商户号后，即可在小程序中使用微信支付收款，以及在小程序后台查看流水、管理资金。
      > 
      >   
      > 小程序支付配置参数`mchid` `appid` `secret`：
      > 
      > > `mchid`: 支付和退款时使用  
          `appid`: 用来生成access_token  
          `secret`: 用来生成access_token
   2. 流程
      1. 在小程序微信公众平台 -功能 - 支付管理入口或调用进件接口 申请商户号。
      2. 完成支付接口（详情请看下文）
      3. 调用wx.requestOrderPayment 拉起支付，完成接口对接


2. 小程序支付接口简单说明
   > http请求方式：POST
   > `https://api.weixin.qq.com/shop/pay/createorder?access_token=xxxxxxxxx`
   
   - 参数说明
   ```php
      {
          "openid": "oTVP50O53a7jgmawAmxKukNlq3XI",
          "combine_trade_no": "P20150806125346",
          "expire_time":1647360558, //int类型  
          "sub_orders":[
            {
              "mchid":"1230000109",
              "amount":100,    //int类型
              "trade_no":"20150806125346",
              "description": "Image形象店 - 深圳腾大 -QQ 公仔"
            }
          ]
      }
   ```
   > `expire_time` 和 `amount` 如果使用string类型，微信接口报错 返回数据格式错误   
   > `combine_trade_no` 和 `trade_no` 长度不能超过32    
   > `description` 字节数不能超过127
   1. 添加小程序支付
      1. 创建`Factory`工厂
      ```php
      <?php
         namespace crmeb\services\wechat;
         use crmeb\services\wechat;

         /**
         * Class Factory.
         *
         * @method static wechat\MiniPayment\Application        MiniPayment(array $config)
         */
         class Factory
         {
            /**
            * @param string $name
            * @param array  $config
            *
            * @return \EasyWeChat\Kernel\ServiceContainer
            */
            public static function make($name, array $config)
            {
               $namespace = \EasyWeChat\Kernel\Support\Str::studly($name);
               $application = "crmeb\\services\\wechat\\{$namespace}\\Application";

               return new $application($config);
            }

            /**
            * Dynamically pass methods to the application.
            *
            * @param string $name
            * @param array  $arguments
            *
            * @return mixed
            */
            public static function __callStatic($name, $arguments)
            {
               return self::make($name, ...$arguments);
            }
         }
        ```
      2. 创建小程序模块
         1. 添加`Application`容器类
         ```php
            <?php
            namespace crmeb\services\wechat\MiniPayment;

            use crmeb\services\wechat\MiniPayment\Payment\WeChatClient;
            use EasyWeChat\MiniProgram\Application as PaymentApplication;

            /**
            * Class Application.
            *
            *
            * @method WeChatClient createMiniOrder(array $params)
            * @method WeChatClient refundMiniOrder(array $params)
            * @property WeChatClient $miniPay
            */

            class Application extends PaymentApplication
            {

               /**
               * @var string[]
               */
               protected $mini_providers = [
                  Payment\ServiceProvider::class
               ];

               /**
               * @param array $config
               * @param array $prepends
               * @param string|null $id
               */
               public function __construct(array $config = [], array $prepends = [], string $id = null)
               {
                  $this->providers = array_merge($this->mini_providers,$this->providers);
                  parent::__construct($config, $prepends, $id);

               }
            }
         ```
         2. 添加`ServiceProvider`服务提供者
         ```php
            <?php
            namespace crmeb\services\wechat\MiniPayment\Payment;

            use Pimple\Container;
            use Pimple\ServiceProviderInterface;

            /**
            * Class ServiceProvider.
            *
            * @author mingyoung <mingyoungcheung@gmail.com>
            */
            class ServiceProvider implements ServiceProviderInterface
            {
               /**
               * @param Container $app
               * @return void
               */
               public function register(Container $app)
               {
                  $app['miniPay'] = function ($app) {
                        return new WeChatClient($app);
                  };
               }
            }
         ```
         3. 添加`WeChatClient`小程序支付类
         ```php
            <?php
            namespace crmeb\services\wechat\MiniPayment\Payment;

            use EasyWeChat\Kernel\BaseClient;

            class WeChatClient extends BaseClient
            {
               private $expire_time = 7000;
               /**
               * 创建订单 支付
               */
               const API_SET_CREATE_ORDER = 'shop/pay/createorder';
               /**
               * 退款
               */
               const API_SET_REFUND_ORDER = 'shop/pay/refundorder';

               /**
               * 支付
               * @param array $params[
               *                      'openid'=>'支付者的openid',
               *                      'out_trade_no'=>'商家合单支付总交易单号',
               *                      'total_fee'=>'支付金额',
               *                      'wx_out_trade_no'=>'商家交易单号',
               *                      'body'=>'商品描述',
               *                      'attach'=>'支付类型',  //product 产品  member 会员
               *                      ]
               * @param $isContract
               * @return mixed
               * @throws \GuzzleHttp\Exception\GuzzleException
               */
               public function createMiniOrder(array $params)
               {
                  $data = [
                        'openid'=>$params['openid'],    // 支付者的openid
                        'combine_trade_no'=>$params['out_trade_no'],  // 商家合单支付总交易单号
                        'expire_time'=>time()+$this->expire_time,
                        'sub_orders'=>[
                           [
                              'mchid'=>$this->app['config']['routine_mch_id'],
                              'amount'=>(int)$params['total_fee'],
                              'trade_no'=>$params['out_trade_no'],
                              'description'=>$params['body'],
                           ]
                        ],
                  ];
                  return $this->httpPostJson(self::API_SET_CREATE_ORDER, $data);
               }

               /**
               * 退款
               * @param array $params[
               *                      'openid'=>'退款者的openid',
               *                      'trade_no'=>'商家交易单号',
               *                      'transaction_id'=>'支付单号',
               *                      'refund_no'=>'商家退款单号',
               *                      'total_amount'=>'订单总金额',
               *                      'refund_amount'=>'退款金额',  //product 产品  member 会员
               *                      ]
               * @return mixed
               * @throws \GuzzleHttp\Exception\GuzzleException
               */
               public function refundMiniOrder(array $params)
               {
                  $data = [
                        'openid'=>$params['openid'],
                        'mchid'=>$this->app['config']['routine_mch_id'],
                        'trade_no'=>$params['trade_no'],
                        'transaction_id'=>$params['transaction_id'],
                        'refund_no'=>$params['refund_no'],
                        'total_amount'=>(int)$params['total_amount'],
                        'refund_amount'=>(int)$params['refund_amount'],
                  ];
                  return $this->httpPostJson(self::API_SET_REFUND_ORDER, $data);
               }
            }
         ```
         4. MiniProgramConfig文件添加小程序商户号
         > 路径：crmeb\services\wechat\config\MiniProgramConfig   
          添加 私有属性 `routineMchId`   
          在`init`方法中给`routineMchId`赋值   
          在`all`方法中返回`routineMchId`  
         ```php
         /**
         * 小程序商户号
         * @var string
         */
         protected $routineMchId;

          /**
         * 初始化
         */
         protected function init()
         {
            $this->routineMchId = $this->routineMchId ?: $this->httpConfig->getConfig('pay.routine_mch_id', '');
         }

         /**
         * 全部
         * @return array
         */
         public function : array
         {
            $this->init();
            return [
                  'routine_mch_id' => $this->routineMchId,
            ];
         }

         ```
         5. 配置文件添加商户号
         > 路径：config\wechat
         在`pay`数组中添加`routine_mch_id`  
         `pay_routine_mchid`是后台小程序支付配置表单添加的字段（商户号为用户输入）
         ```php
         //支付
         'pay' => [
            'routine_mch_id' => 'pay_routine_mchid',//小程序商户号
         ]
         ```
         6. 使用
         > 需要兼容之前支付，应为小程序支付只是部分用户灰度升级而来，所以需要判断走之前的微信支付还是现在新的小程序支付  
         pro版修改方法：
         在后台支付配置里面添加字段`is_routine`，由用户自行判断，是否打开，打开是需要填写小程序支付管理中的商户号
         >> 说明：
         在 `crmeb\services\wechat\Payment` 添加新的方法`miniApplication`  
         `$config = $make->all();` 获取支付所需参数  
         `miniFactory::MiniPayment($config);` 通过工厂去实例化我们创建的`Application`容器类，   
         容器里面会去实例化所以要的服务类`ServiceProvider`，           
         在服务类里面调用了 `Container` 类的 `register`方法；  
         `$app['miniPay']`语法的赋值行为会执行`Pimple\Container`类的`offsetSet`方法。
         ```php
         /**
         * @return \crmeb\services\wechat\MiniPayment\Application
         */
         public function miniApplication()
         {
            $request = request();
            $accessEnd = $this->getAuthAccessEnd($request);
            if($accessEnd !== 'mini')
            {
                  throw new PayException('支付方式错误，请刷新后重试！');
            }
            /** @var MiniProgramConfig $make */
            $make = app()->make(MiniProgramConfig::class);
            $config = $make->all();
            if (!isset($this->application[$accessEnd])) {
                  $this->application[$accessEnd] = miniFactory::MiniPayment($config);
                  $this->application[$accessEnd]['guzzle_handler'] = SwooleHandler::class;
                  $this->application[$accessEnd]->rebind('request', new Request($request->get(), $request->post(), [], [], [], $request->server(), $request->getContent()));
            }
            return $this->application[$accessEnd];
         }

         /**
         * 生成支付订单对象
         * @param $openid
         * @param $out_trade_no
         * @param $total_fee
         * @param $attach
         * @param $body
         * @param string $detail
         * @param $trade_type
         * @param array $options
         * @return array|Collection|object|ResponseInterface|string
         * @throws InvalidConfigException
         * @throws InvalidArgumentException
         * @throws GuzzleException
         */
         public static function paymentMiniOrder($openid, $out_trade_no, $total_fee, $attach, $body, $detail = '', $trade_type = 'JSAPI', array $options = [])
         {
            $total_fee = bcmul($total_fee, 100, 0);
            $order = array_merge(compact('out_trade_no', 'total_fee', 'attach', 'body', 'detail', 'trade_type'), $options);
            if (!is_null($openid)) $order['openid'] = $openid;
            if ($order['detail'] == '') unset($order['detail']);
            $order['spbill_create_ip'] = request()->ip();
            $result = self::instance()->miniApplication()->miniPay->createMiniOrder($order);
            self::logger('生成支付订单对象', compact('order'), $result);
            if ($result['errcode'] == '0') {
                  return $result;
            } else {
                  throw new PayException('微信支付错误返回：' . $result['errmsg']);
            }
         }
         ```
3. 前端修改
> 小程序升级为支付管理后需要requestOrderPayment拉起支付  
 兼容之前版本，先判断requestOrderPayment是否可用，如果可用则使用requestOrderPayment，如果不能用则使用requestPayment
```js
    // 判断requestOrderPayment是否可用
   let mp_pay_name=''
   if(uni.requestOrderPayment){
     mp_pay_name='requestOrderPayment'
   }else{
     mp_pay_name='requestPayment'
   }
   // 发起微信支付
    uni[mp_pay_name]({...});
```
