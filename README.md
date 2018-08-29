new version of Udp:
// 添加依赖<br>
================
1.工程gradle.allprojects.repositories添加<br>
```
 maven { url "https://github.com/slinphonesdk/extension/raw/master" }
 maven { url "https://github.com/slinphonesdk/common/raw/master" }
 maven { url "https://github.com/slinphonesdk/lib/raw/master" }
```
2.项目gradle.dependencies添加<br>
```
 implementation 'com.udp:extension:1.1.2'
```
*打包需知：common打包后，依赖至extension，再打extension包*
-------------------------------
方法列表
=======
```
// TODO: 初始化方法
public ExPhoneManager(Context context, String serverIP, String serverPort)

// TODO: sip 注册状态 通话状态监听
public void addListener(ExPhoneListener exPhoneListener)

// TODO: 响应收到 - 收到信息主机消息后,将信息标识返回
public void response(String msgFlag, InetAddress fromAddress)

// TODO: 注册成为移动分机
public void registerMobile(RUdpCallBack rUdpCallBack)

// TODO: 取消注册为非移动分机
public void unregisterMobile(RUdpCallBack rUdpCallBack)

// TODO: 卫浴分机呼叫
public void bathroomCall(RUdpCallBack rUdpCallBack)

// TODO: 取消呼叫
public void cancelCall(String sessionID, RUdpCallBack rUdpCallBack)

// TODO: 床头发起呼叫
public void call(String nurseLevel, RUdpCallBack rUdpCallBack)

// TODO: 拒绝通话
public void refusingToAnswer(String sessionID, RUdpCallBack rUdpCallBack)

// TODO: 接听
public final void acceptCall(String sessionID, final RUdpCallBack rUdpCallBack)

// TODO: 挂断
public final void hangup(String sessionID, final RUdpCallBack rUdpCallBack)

// TODO: sip 通话挂断 - 当医护分机主动发起呼叫时，无sessionID时调用
public final void sipHangup()

// TODO: 向服务器注册
public void registerToServer()

// TODO: 向信息主机注册
public void registerToInfoMaster(final String sipNumber, DeviceType deviceType, String uuid, final RUdpCallBack rUdpCallBack)

// TODO: 停止发送心跳 - 收到IP冲突信息后调用
public void stopSendHeart()

// TODO: 销毁
public void destroy()

```
DEMO
```
public class RUdpMainActivity extends AppCompatActivity {

    private TextView textView;
    private TextView regTv;
    private String sessionID;
    private String recSessionID;
    private String msg = "";
    private String regMsg = "";
    private ArrayList<String> sessionList = new ArrayList<>();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_rudp_main);
        setupUI();
        setupEx();
        updateUI();
    }

    private ExPhoneManager exPhoneManager;
    private void setupEx() {

        // 1. exPhoneManager init
        exPhoneManager = new ExPhoneManager(this);

        // 2.sip registration State and call state
        exPhoneManager.addListener(new ExPhoneListener() {
            @Override
            public void registrationState(String s, String s1) {
                regMsg = " 注册状态："+s+" "+s1;
                updateUI();
            }

            @Override
            public void callState(String s, int i, String s1) {
                msg += s+" "+i+" "+s1;
                updateUI();
            }
        });

        // 3.register to info master
        exPhoneManager.registerToInfoMaster("8807001", DeviceType.bedHeader, "bed1",new RUdpCallBack() {

            @Override
            public void onFailure(String callMsg, IOException e) {
                msg += "\n 注册超时";
                updateUI();
            }

            @Override
            public void onResponse(String CallMsg, String response) throws IOException {
                sendResponseModel(response);
            }
        });

        // 4.receive from info master
        exPhoneManager.addUdpListener(new RUdpListener() {
            @Override
            public void dataResponse(String msgFlag, String result, InetAddress fromAddress) {

                try {
                    receiveResponseModel(msgFlag,result,fromAddress);
                } catch (InvalidProtocolBufferException e) {
                    e.printStackTrace();
                }
            }
        });
    }

    private void setupUI() {
        regTv = findViewById(R.id.reg_state_tv);
        textView = findViewById(R.id.textView);
    }

    private void updateUI() {
        if (msg.length() > 400) msg = "";
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                regTv.setText(regMsg);
                textView.setText(msg);
            }
        });
    }

    // TODO: 接收响应数据处理
    private void receiveResponseModel(String msgFlag, String result, InetAddress fromAddress) throws InvalidProtocolBufferException {

        ParamsModel paramsModel = ParamsModel.getParamsHeader(result);
        if (paramsModel == null) return;

        if (paramsModel.paramsHeader.msgCommand == MsgCommand.receiveBroadcastSingle) { // 配对分机收到会话
            exPhoneManager.response(msgFlag,fromAddress);
            recSessionID = paramsModel.body.getSessionID();
            msg += "\n 配对分机收到分机呼叫: "+recSessionID;
            updateUI();
        }
        else if (paramsModel.paramsHeader.msgCommand == MsgCommand.refusingToanswer) { // 呼叫被拒绝
            exPhoneManager.response(msgFlag,fromAddress);
            msg += "\n 发起呼叫被拒绝";
            updateUI();
            recSessionID = "";
            sessionID = "";
        }
        else if (paramsModel.paramsHeader.msgCommand == MsgCommand.called) { // 呼叫被接听
            exPhoneManager.response(msgFlag,fromAddress);
            msg += "\n 发起呼叫即将被接听";
            updateUI();
        }
        else if (paramsModel.paramsHeader.msgCommand == MsgCommand.hanguped) { // 呼叫被挂断
            exPhoneManager.response(msgFlag, fromAddress);
            exPhoneManager.sipHangup();
            msg += "\n 发起呼叫被挂断";
            updateUI();
            recSessionID = "";
            sessionID = "";
        }
        else if (paramsModel.paramsHeader.msgCommand == MsgCommand.receiveBroadcast) { // 医护分机收到会话列表
            exPhoneManager.response(msgFlag,fromAddress);
            sessionList.clear();
            if (paramsModel.body.getListsCount() > 0 ) {
                for (int i = 0; i < paramsModel.body.getListsCount(); i++) {
                    String sessionID = paramsModel.body.getLists(i);
                    sessionList.add(sessionID);
                    if (i == 0) {
                        RUdpMainActivity.this.recSessionID = sessionID;
                    }
                }
            }
            msg += "\n" + sessionList.toString();
            updateUI();
        } else if (paramsModel.paramsHeader.msgCommand == MsgCommand.online) { // 医护分机收到某分机在线消息
            exPhoneManager.response(msgFlag,fromAddress);
            msg += "\n"+paramsModel.paramsHeader.sip + " from address: "+paramsModel.paramsHeader.uuid + "已在线";
            updateUI();
        } else if (paramsModel.paramsHeader.msgCommand == MsgCommand.offline) { // 医护分机收到某分机在线消息
            exPhoneManager.response(msgFlag,fromAddress);
            msg += "\n"+paramsModel.paramsHeader.sip + " from address: "+paramsModel.paramsHeader.uuid + "已离线";
            updateUI();
        } else if (paramsModel.paramsHeader.msgCommand == MsgCommand.SIPConflict) { // 有冲突
            exPhoneManager.stopSendHeart();
            exPhoneManager.response(msgFlag,fromAddress);
            msg = "SIP 设置有冲突";
            updateUI();
        }
    }


    // TODO: 发送响应数据处理
    private void sendResponseModel(String result) throws InvalidProtocolBufferException {

        ParamsModel paramsModel = ParamsModel.getParamsHeader(result);
        if (paramsModel == null) return;

        if (paramsModel.paramsHeader.msgCommand == MsgCommand.callRecSessionID) { // from 信息主机
            sessionID = paramsModel.body.getSessionID();
            msg += "\n"+" 已发起呼叫 会话ID: "+sessionID;
            updateUI();
        } else if (paramsModel.paramsHeader.msgCommand == MsgCommand.registerSuccess) {
            exPhoneManager.registerToServer();
            msg += "\n"+" 注册成功";
            updateUI();
        } else if (paramsModel.paramsHeader.msgCommand == MsgCommand.registerFailed) {
            msg += "\n"+" 注册失败 "+paramsModel.paramsHeader.uuid;
            updateUI();
        }
    }

    public void onClick(View view) {

        int id = view.getId();
        switch (id) {
            case R.id.call:
                exPhoneManager.call("1",new RUdpCallBack() {
                    @Override
                    public void onFailure(String callMsg, IOException e) {
                        msg += "\n"+" 呼叫超时";
                        updateUI();
                    }

                    @Override
                    public void onResponse(String CallMsg, String response) throws IOException {
                        sendResponseModel(response);
                    }
                });
                break;
            case R.id.cancel:
                exPhoneManager.cancelCall(sessionID, new RUdpCallBack() {
                    @Override
                    public void onFailure(String callMsg, IOException e) {
                        msg += "\n 取消呼叫超时";
                        updateUI();
                    }

                    @Override
                    public void onResponse(String CallMsg, String response) {
                        msg += "\n 取消呼叫成功";
                        updateUI();
                    }
                });
                break;
            case R.id.receive:
                if (recSessionID.length() > 0)
                    exPhoneManager.acceptCall(recSessionID, new RUdpCallBack() {
                        @Override
                        public void onFailure(String callMsg, IOException e) {
                            msg += "\n 配对分机：接听"+recSessionID+"超时";
                            updateUI();
                        }

                        @Override
                        public void onResponse(String CallMsg, String response) {
                            msg += "\n 配对分机：接听"+recSessionID+"成功 马上发起sip呼叫";
                            updateUI();
                        }
                    });
                break;
            case R.id.hangup:
                String currentSessionID = "";
                if (recSessionID.length() > 0) currentSessionID = recSessionID;
                if (sessionID.length() > 0) currentSessionID = sessionID;

                final String curSessionID = currentSessionID;
                if (curSessionID.length() > 0)
                    exPhoneManager.hangup(currentSessionID, new RUdpCallBack() {
                        @Override
                        public void onFailure(String callMsg, IOException e) {
                            msg += "\n 挂断: "+curSessionID+"超时";
                            updateUI();
                        }

                        @Override
                        public void onResponse(String CallMsg, String response) {
                            msg += "\n 挂断: "+curSessionID+"成功";
                            updateUI();
                            recSessionID = "";
                            sessionID = "";
                        }
                    });
                break;
            case R.id.refuse:
                if (recSessionID.length() > 0)
                    exPhoneManager.refusingToAnswer(recSessionID, new RUdpCallBack() {
                        @Override
                        public void onFailure(String callMsg, IOException e) {
                            msg += "\n 拒绝: "+recSessionID+"超时";
                            updateUI();
                        }

                        @Override
                        public void onResponse(String CallMsg, String response) {
                            msg += "\n 拒绝: "+recSessionID+"成功";
                            updateUI();
                            recSessionID = "";
                            sessionID = "";
                        }
                    });
                break;
            case R.id.reg:
                exPhoneManager.registerMobile(new RUdpCallBack() {
                    @Override
                    public void onFailure(String callMsg, IOException e) {
                        msg += "\n 注册配对超时";
                        updateUI();
                    }

                    @Override
                    public void onResponse(String CallMsg, String response) {
                        msg += "\n 注册配对成功";
                        updateUI();
                    }
                });

                break;
            case R.id.reg_cancel:
                exPhoneManager.unregisterMobile(new RUdpCallBack() {
                    @Override
                    public void onFailure(String callMsg, IOException e) {
                        msg += "\n 取消配对超时";
                        updateUI();
                    }

                    @Override
                    public void onResponse(String CallMsg, String response) {
                        msg += "\n 取消配对成功";
                        updateUI();
                    }
                });
                break;
            case R.id.reReg:
                exPhoneManager.registerToInfoMaster("9908001", DeviceType.bedHeader, "bed2", new RUdpCallBack() {
                    @Override
                    public void onFailure(String callMsg, IOException e) {
                        msg += "\n 重新注册超时";
                        updateUI();
                    }

                    @Override
                    public void onResponse(String CallMsg, String response) throws InvalidProtocolBufferException {
                        sendResponseModel(response);
                    }
                });
                break;
        }
    }

    @Override
    protected void onDestroy() {
        exPhoneManager.destroy();
        exPhoneManager = null;
        super.onDestroy();
    }

}
```


