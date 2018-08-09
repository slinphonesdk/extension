new:
exPhoneManager.reConfig("1508001", DeviceType.bedHeader, "bed14", new ExPhoneManager.RegisterListener() {
            @Override
            public void state(Boolean bool) {
                Log.e("REG", bool.toString());
            }
        });

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
医护分机
--------
```
private ArrayList<String> list = new ArrayList<>();
    private void exphoneSetup() {
	// 初始化方法
        ExphoneManager.init(this,"100000", DeviceType.treatAndNurse, "mas");
// 添加监听        
ExphoneManager.getInstance().addListener(new ExphoneListener() {

// 收到信息           
 @Override
            public void msgCallBack(ParamsHeader paramsHeader, Msgcommand msgcommand, final CMC.CMCRequestParam.Body body) {
                Log.e("ppt", "exphone ----");
                sessionID = body.getSessionID();
                if (paramsHeader.msgcommand == Msgcommand.receiveBroadcast) {
                    Log.e("ppt","广播消息(医护分机)：list:");
                    list.clear();
                    if (body.getListsCount() > 0 ) {
                        for (int i = 0; i < body.getListsCount(); i++) {
                            list.add(body.getLists(i));
                        }
                    }
                    if (list.size() > 0)
                        sessionID = list.get(0);

                    textView.post(new Runnable() {
                        @Override
                        public void run() {
                            textView.setText(list.toString());
                        }
                    });

                }
            }

	   // 发起呼叫得到的会话id，在主动取消通话时需要传
            @Override
            public void sessionStatus(String sessionID) { }

	   // sip 自动应答后会调用此方法，更新界面
            @Override
            public void sipCallIncomingReceived() {

            }

	    // 通话结束 上传呼叫记录
            @Override
            public void sipCallEnd() {
            }
        });
    }


    // 发起呼叫 
    public void startCall(View v) {
        ExphoneManager.getInstance().call();
    }

// 接听呼叫
    public void receive(View v) {
        if (sessionID != null) {
            ExphoneManager.getInstance().acceptCall(sessionID);
        }
    }

// 挂断
    public void hangup(View v) {
        if (sessionID != null) {
            ExphoneManager.getInstance().hangupOrCancelCall(sessionID);
            removeFromList();
        }

        textView.post(new Runnable() {
            @Override
            public void run() {
                if (list != null) {
                    textView.setText(list.toString());
                }
            }
        });
    }

// 拒绝
    public void refuse(View v) {
        if (sessionID != null) {
            ExphoneManager.getInstance().refusingToAnswer(sessionID);
            removeFromList();
        }

        textView.post(new Runnable() {
            @Override
            public void run() {
                textView.setText(list.toString());
            }
        });
    }

    private void removeFromList() {
        if (sessionID != null && list != null) {

            if (list.size() > 0) {
                if (list.contains(sessionID)) {
                    list.remove(sessionID);
                }
                if (list.size() > 0) {
                    sessionID = list.get(0);
                }
                else {
                    sessionID = "";
                }
            }
        }
    }
```
床头分机
--------
```
 public void register(View v) {
        ExphoneManager.getInstance().registerOfTreatAndNurse();
        textView.post(new Runnable() {
            @Override
            public void run() {
                textView.setText("移动-医护分机-已注册");
            }
        });
    }

    public void unregister(View v) {
        ExphoneManager.getInstance().unregister();
        textView.post(new Runnable() {
            @Override
            public void run() {
                textView.setText("移动-医护分机-已取消");
            }
        });
    }

    private void exphoneSetup() {
        ExphoneManager.init(this,"1000", DeviceType.bedHeader, "bed1");
        ExphoneManager.getInstance().addListener(new ExphoneListener() {
            @Override
            public void msgCallBack(ParamsHeader paramsHeader, Msgcommand msgcommand, final CMC.CMCRequestParam.Body body) {
                Log.e("ppt", "exphone ----");
                if (paramsHeader.msgcommand == Msgcommand.refusingToanswer) {
                    sessionID = "";
                    Log.e("ppt","此会话被拒绝：sessionID:"+body.getSessionID());
                    textView.post(new Runnable() {
                        @Override
                        public void run() {
                            textView.setText("此会话被拒绝：sessionID:"+body.getSessionID());
                        }
                    });
                }
                if (paramsHeader.msgcommand == Msgcommand.callRecSessionID) {//from info. master
                    sessionID = body.getSessionID();
                    Log.e("ppt","呼叫得到的：sessionID:"+body.getSessionID());
                    textView.post(new Runnable() {
                        @Override
                        public void run() {
                            textView.setText("呼叫得到的：sessionID:"+body.getSessionID());
                        }
                    });
                } else if (paramsHeader.msgCommand == MsgCommand.cancelCallSuccess) {
                    Log.e("ppt","取消呼叫成功");
                }
                else if (paramsHeader.msgCommand == MsgCommand.registerSuccess) {
                    Log.e("ppt","注册成功");
                }
                else if (paramsHeader.msgCommand == MsgCommand.registerFailed) {
                    Log.e("ppt", "注册失败");
                }
            }

            @Override
            public void sessionStatus(final String sessionID) {
                Log.e("ppt","sessionID: "+sessionID);
                MainActivityDemo.this.sessionID = sessionID;
                textView.post(new Runnable() {
                    @Override
                    public void run() {
                        textView.setText("sessionID: "+sessionID);
                    }
                });
            }

            @Override
            public void sipCallIncomingReceived() {

            }

            @Override
            public void sipCallEnd() { }
        });
    }



    public void startCall(View v) {
        ExphoneManager.getInstance().call();
    }

    public void receive(View v) {
        if (sessionID != null) {
            ExphoneManager.getInstance().acceptCall(sessionID);
        }
    }

    public void hangup(View v) {
        if (sessionID != null) {
            ExphoneManager.getInstance().hangupOrCancelCall(sessionID);
            textView.post(new Runnable() {
                @Override
                public void run() {
                    textView.setText("");
                }
            });
        }
    }

    public void refuse(View v) {
        if (sessionID != null) {
            ExphoneManager.getInstance().refusingToAnswer(sessionID);
        }
    }
```
门口分机
--------
```
 private void exPhoneSetup() {
        ExPhoneManager.init(this,"1108", DeviceType.doorSide, "cbo");
        ExPhoneManager.getInstance().addListener(new ExPhoneListener() {
            @Override
            public void msgCallBack(ParamsHeader paramsHeader, MsgCommand msgCommand, final CMC.CMCRequestParam.Body body) { }

            @Override
            public void sessionStatus(final String sessionID) {
                Log.e("ppt","sessionID: "+sessionID);
                MainActivityDemo.this.sessionID = sessionID;
                textView.post(new Runnable() {
                    @Override
                    public void run() {
                        textView.setText("sessionID: "+sessionID);
                    }
                });
            }

            @Override
            public void sipCallIncomingReceived() { }

            @Override
            public void sipCallEnd(){
            }

            @Override
            public void keepLineState(PhoneLineState state, String sip) {

            }
        });
    }

    public void register(View v) {
        ExPhoneManager.getInstance().registerOfTreatAndNurse();
        textView.post(new Runnable() {
            @Override
            public void run() {
                textView.setText("移动-医护分机-已注册");
            }
        });
    }

    public void unregister(View v) {
        ExPhoneManager.getInstance().unregister();
        textView.post(new Runnable() {
            @Override
            public void run() {
                textView.setText("移动-医护分机-已取消");
            }
        });
    }
```
