# android-bluetooth
本篇文章不是原创作品 是我在csdn上看见的不错demo ，传到github上是为了让更多的人能够使用，
demo直接可以跑得同 功能齐全，代码简单

Android 的blt仅仅支持api 18 android4.3以上，有的功能甚至需要api 19 android4.4； 
所以我们在做blt项目之前一定要清楚可用的版本范围。

我要讲述的是打开blt大门的操作。这些操作就是如何打开blt、如何搜索到其他设备、如何配对选中设备、如何通过mac地址连接之前已经配对过的设备以及连接成功的两个（或一对多个）设备如何通讯。

在学习blt知识前要先搞清楚，blt是如何让两个设备之间通讯的。我们先来看一下基本步骤：

1，打开blt。 
——1）权限 
——2）监听设备是否打开blt 
——3）操作，打开或者关闭blt

2，搜索附近设备&让附近设备搜索到 
——1）让自身设备可以被附近搜索到，时间最长300s；（随着api的逐渐升级可能不止300s，做了一定的优化处理） 
——2）搜索附近可连接的设备；过时的api在接口中回调搜索结果，新的api则要求用广播接收搜索结果。（我们这里使用广播接收结果）

3，与目标设备配对 
——1）对于目标设备进行配对，android系统本身会完成配对动作，并且将已经成功配对的设备的mac地址保存起来，以供下次进行自动配对使用。 
——2）对进行配对过的设备自动配对。这个动作也是系统帮我们完成的，我们只需要根据系统给我们的状态来判断这个是否已经配对就行了。

4，与成功配对的设备进行连接 
——1）如果要对成功配对的设备进行连接，则必须先建立服务器端。服务器端建立只有会线程阻塞等待客户端的连接。 
——2）确保建立了服务器端之后再建立客户端。客户端的连接过程也是线程阻塞的。知道连接成功后，服务器端收到消息，则表示配对设备已连接成功。

5，注意事项： 
——1）配对成功过的设备无需再次配对，只需要从系统中拿到mac地址进行连接。这个根据系统返回的状态值去判断。 
——2）搜索附近设备的操作是一个消耗内存的动作，我们务必在连接设备‘之前‘停止搜索。 
——3）设备的配对成功不代表连接成功。配对和连接是两回事且配对在连接动作之前。 
——4）当我们的程序明确指出“不需要blt操作了”的时候，及时反注册blt广播，停止blt连接，注销蓝牙对象。而这个操作例如：程序退出、用户无操作超时、逻辑不需要blt的连接了。

以上就是一个蓝牙连接成功之后可以正常传输信息的步骤。 
接下来我们再来看看相应的步骤在代码逻辑中的代表。

1，权限。

<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />

2，获得蓝牙管理对象。BluetoothManager的主要作用是从中得到我们需要的对象

//@TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR2)
//首先获取BluetoothManager
 BluetoothManager bluetoothManager=(BluetoothManager) context.getService(Context.BLUETOOTH_SERVICE);

3，获得蓝牙适配器。蓝牙适配器是我们操作蓝牙的主要对象，可以从中获得配对过的蓝牙集合，可以获得蓝牙传输对象等等

//获取BluetoothAdapter
if (bluetoothManager != null)
    BluetoothAdapter mBluetoothAdapter = bluetoothManager.getAdapter();

3，注册广播来接收蓝牙配对信息。在蓝牙开启前调用

// 用BroadcastReceiver来取得搜索结果
        IntentFilter intent = new IntentFilter();
        intent.addAction(BluetoothDevice.ACTION_FOUND);//搜索发现设备
        intent.addAction(BluetoothDevice.ACTION_BOND_STATE_CHANGED);//状态改变
        intent.addAction(BluetoothAdapter.ACTION_SCAN_MODE_CHANGED);//行动扫描模式改变了
        intent.addAction(BluetoothAdapter.ACTION_STATE_CHANGED);//动作状态发生了变化
        context.registerReceiver(searchDevices, intent);
 /**
     * 蓝牙接收广播
     */
    private BroadcastReceiver searchDevices = new BroadcastReceiver() {
        //接收
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            Bundle b = intent.getExtras();
            Object[] lstName = b.keySet().toArray();

            // 显示所有收到的消息及其细节
            for (int i = 0; i < lstName.length; i++) {
                String keyName = lstName[i].toString();
                Log.e("bluetooth", keyName + ">>>" + String.valueOf(b.get(keyName)));
            }
            BluetoothDevice device;
            // 搜索发现设备时，取得设备的信息；注意，这里有可能重复搜索同一设备
            if (BluetoothDevice.ACTION_FOUND.equals(action)) {
                device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
                onRegisterBltReceiver.onBluetoothDevice(device);
            }
            //状态改变时
            else if (BluetoothDevice.ACTION_BOND_STATE_CHANGED.equals(action)) {
                device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
                switch (device.getBondState()) {
                    case BluetoothDevice.BOND_BONDING://正在配对
                        Log.d("BlueToothTestActivity", "正在配对......");
                        onRegisterBltReceiver.onBltIng(device);
                        break;
                    case BluetoothDevice.BOND_BONDED://配对结束
                        Log.d("BlueToothTestActivity", "完成配对");
                        onRegisterBltReceiver.onBltEnd(device);
                        break;
                    case BluetoothDevice.BOND_NONE://取消配对/未配对
                        Log.d("BlueToothTestActivity", "取消配对");
                        onRegisterBltReceiver.onBltNone(device);
                    default:
                        break;
                }
            }
        }
    };

4，反注册广播和清楚蓝牙连接。在不需要使用蓝牙的时候调用

/**
     * 反注册广播取消蓝牙的配对
     *
     * @param context
     */
    public void unregisterReceiver(Context context) {
        context.unregisterReceiver(searchDevices);
        if (mBluetoothAdapter != null)
            mBluetoothAdapter.cancelDiscovery();
    }

5，判断当前设备是否支持蓝牙，如果支持则打开蓝牙

   /**
     * 判断是否支持蓝牙，并打开蓝牙
     * 获取到BluetoothAdapter之后，还需要判断是否支持蓝牙，以及蓝牙是否打开。
     * 如果没打开，需要让用户打开蓝牙：
     */
    public void checkBleDevice(Context context) {
        if (mBluetoothAdapter != null) {
            if (!mBluetoothAdapter.isEnabled()) {
                Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
                enableBtIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                context.startActivity(enableBtIntent);
            }
        } else {
            Log.i("blueTooth", "该手机不支持蓝牙");
        }
    }

6，搜索蓝牙。搜索后添加到自定义的集合中以供自身程序的逻辑使用。注意：搜索到的结果会在广播中回调过来，这里不做回调。若使用过时的接口，则在当前逻辑里回调结果。我们这里使用广播，不建议使用过时的方法。

/**
     * 搜索蓝牙设备
     * 通过调用BluetoothAdapter的startLeScan()搜索BLE设备。
     * 调用此方法时需要传入 BluetoothAdapter.LeScanCallback参数。
     * 因此你需要实现 BluetoothAdapter.LeScanCallback接口，BLE设备的搜索结果将通过这个callback返回。
     * <p/>
     * 由于搜索需要尽量减少功耗，因此在实际使用时需要注意：
     * 1、当找到对应的设备后，立即停止扫描；
     * 2、不要循环搜索设备，为每次搜索设置适合的时间限制。避免设备不在可用范围的时候持续不停扫描，消耗电量。
     * <p/>
     * 如果你只需要搜索指定UUID的外设，你可以调用 startLeScan(UUID[], BluetoothAdapter.LeScanCallback)方法。
     * 其中UUID数组指定你的应用程序所支持的GATT Services的UUID。
     * <p/>
     * 注意：搜索时，你只能搜索传统蓝牙设备或者BLE设备，两者完全独立，不可同时被搜索。
     */
    private boolean startSearthBltDevice(Context context) {
        //开始搜索设备，当搜索到一个设备的时候就应该将它添加到设备集合中，保存起来
        checkBleDevice(context);
        //如果当前发现了新的设备，则停止继续扫描，当前扫描到的新设备会通过广播推向新的逻辑
        if (getmBluetoothAdapter().isDiscovering())
            stopSearthBltDevice();
        Log.i("bluetooth", "本机蓝牙地址：" + getmBluetoothAdapter().getAddress());
        //开始搜索
        mBluetoothAdapter.startDiscovery();
        //这里的true并不是代表搜索到了设备，而是表示搜索成功开始。
        return true;
    }

7，停止搜索蓝牙设备

public boolean stopSearthBltDevice() {
        //暂停搜索设备
        if(mBluetoothAdapter!=null)
        return mBluetoothAdapter.cancelDiscovery();
    }

8,连接蓝牙。注意：连接蓝牙的前提是我们已经建立好了蓝牙服务器端。所以，这里我们先建立蓝牙服务器
——1）先建立蓝牙服务器端

/**
     * 这个操作应该放在子线程中，因为存在线程阻塞的问题
     */
    public void run(Handler handler) {
        //服务器端的bltsocket需要传入uuid和一个独立存在的字符串，以便验证，通常使用包名的形式
        BluetoothServerSocket  bluetoothServerSocket = tmBluetoothAdapter.listenUsingRfcommWithServiceRecord("com.bluetooth.demo", BltContant.SPP_UUID);
        while (true) {
            try {
                //注意，当accept()返回BluetoothSocket时，socket已经连接了，因此不应该调用connect方法。
                //这里会线程阻塞，直到有蓝牙设备链接进来才会往下走
                socket = getBluetoothServerSocket().accept();
                if (socket != null) {
                    BltAppliaction.bluetoothSocket = socket;
                    //回调结果通知
                    Message message = new Message();
                    message.what = 3;
                    message.obj = socket.getRemoteDevice();
                    handler.sendMessage(message);
                    //如果你的蓝牙设备只是一对一的连接，则执行以下代码
                    getBluetoothServerSocket().close();
                    //如果你的蓝牙设备是一对多的，则应该调用break；跳出循环
                    //break;
                }
            } catch (IOException e) {
                try {
                    getBluetoothServerSocket().close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
                break;
            }
        }
    }

——2）在蓝牙服务器建立之后，再进行连接蓝牙的操作。

 /**
     * 尝试连接一个设备，子线程中完成，因为会线程阻塞
     *
     * @param btDev 蓝牙设备对象
     * @param handler 结果回调事件
     * @return
     */
    private void connect(BluetoothDevice btDev, Handler handler) {
        try {
            //通过和服务器协商的uuid来进行连接
            mBluetoothSocket = btDev.createRfcommSocketToServiceRecord(BltContant.SPP_UUID);
            if (mBluetoothSocket != null)
                //全局只有一个bluetooth，所以我们可以将这个socket对象保存在appliaction中
                BltAppliaction.bluetoothSocket = mBluetoothSocket;
            //通过反射得到bltSocket对象，与uuid进行连接得到的结果一样，但这里不提倡用反射的方法
            //mBluetoothSocket = (BluetoothSocket) btDev.getClass().getMethod("createRfcommSocket", new Class[]{int.class}).invoke(btDev, 1);
            Log.d("blueTooth", "开始连接...");
            //在建立之前调用
            if (getmBluetoothAdapter().isDiscovering())
                //停止搜索
                getmBluetoothAdapter().cancelDiscovery();
            //如果当前socket处于非连接状态则调用连接
            if (!getmBluetoothSocket().isConnected()) {
                //你应当确保在调用connect()时设备没有执行搜索设备的操作。
                // 如果搜索设备也在同时进行，那么将会显著地降低连接速率，并很大程度上会连接失败。
                getmBluetoothSocket().connect();
            }
            Log.d("blueTooth", "已经链接");
            if (handler == null) return;
            //结果回调
            Message message = new Message();
            message.what = 4;
            message.obj = btDev;
            handler.sendMessage(message);
        } catch (Exception e) {
            Log.e("blueTooth", "...链接失败");
            try {
                getmBluetoothSocket().close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
            e.printStackTrace();
        }
    }

9，自动连接以往配对成功的设备。注意：连接的前提是服务器端已开启，若没开启则进行8.1的操作

/**
     * 尝试配对和连接
     *
     * @param btDev
     */
    public void createBond(BluetoothDevice btDev, Handler handler) {
        if (btDev.getBondState() == BluetoothDevice.BOND_NONE) {
            //如果这个设备取消了配对，则尝试配对
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
                btDev.createBond();
            }
        } else if (btDev.getBondState() == BluetoothDevice.BOND_BONDED) {
            //如果这个设备已经配对完成，则尝试连接
            connect(btDev, handler);
        }
    }

 /**
     * 获得系统保存的配对成功过的设备，并尝试连接
     */
    public void getBltList() {
        if (getmBluetoothAdapter() == null) return;
        //获得已配对的远程蓝牙设备的集合
        Set<BluetoothDevice> devices = getmBluetoothAdapter().getBondedDevices();
        if (devices.size() > 0) {
            for (Iterator<BluetoothDevice> it = devices.iterator(); it.hasNext(); ) {
                BluetoothDevice device = it.next();
                //自动连接已有蓝牙设备
                createBond(device, null);
            }
        }
    }

注意：外界只需要调用getBltList();方法即可进行自动连接。

10，输入mac地址自动连接设备。前提是系统原来连接过该地址。

 /**
     * 输入mac地址进行自动配对
     * 前提是系统保存了该地址的对象
     *
     * @param address
     */
    public void autoConnect(String address, Handler handler) {
        if (getmBluetoothAdapter().isDiscovering()) getmBluetoothAdapter().cancelDiscovery();
        BluetoothDevice btDev = getmBluetoothAdapter().getRemoteDevice(address);
        connect(btDev, handler);
    }
蓝牙连接状态。用于我们判断该蓝牙设备是出于：未配对还是配对未连接还是已连接状态

public String bltStatus(int status) {
        String a = "未知状态";
        switch (status) {
            case BluetoothDevice.BOND_BONDING:
                a = "连接中";
                break;
            case BluetoothDevice.BOND_BONDED:
                a = "连接完成";
                break;
            case BluetoothDevice.BOND_NONE:
                a = "未连接/取消连接";
                break;
        }
        return a;
    }
蓝牙点击事件。包括蓝牙的打开，关闭，被搜索，断开连接

/**
     * 蓝牙操作事件
     *
     * @param context
     * @param status
     */
    public void clickBlt(Context context, int status) {
        switch (status) {
            case BltContant.BLUE_TOOTH_SEARTH://搜索蓝牙设备，在BroadcastReceiver显示结果
                startSearthBltDevice(context);
                break;
            case BltContant.BLUE_TOOTH_OPEN://本机蓝牙启用
                if (getmBluetoothAdapter() != null)
                    getmBluetoothAdapter().enable();//启用
                break;
            case BltContant.BLUE_TOOTH_CLOSE://本机蓝牙禁用
                if (getmBluetoothAdapter() != null)
                    getmBluetoothAdapter().disable();//禁用
                break;
            case BltContant.BLUE_TOOTH_MY_SEARTH://本机蓝牙可以在300s内被搜索到
                Intent discoverableIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_DISCOVERABLE);
                discoverableIntent.putExtra(BluetoothAdapter.EXTRA_DISCOVERABLE_DURATION, 300);
                context.startActivity(discoverableIntent);
                break;
            case BltContant.BLUE_TOOTH_CLEAR://本机蓝牙关闭当前连接
                try {
                    if (getmBluetoothSocket() != null)
                        getmBluetoothSocket().close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                break;
        }
    }

到此，蓝牙从打开到连接的方法都写完了。最后我们来总结一下。 
当我们获得到了bluetoothsocket对象的时候，我们就可以像使用socket编程一样，让两个蓝牙之间传输数据。甚至可以在程序内部监听蓝牙耳机的暂停/播放/音量键等的点击事件。

具体的蓝牙操作，我将放在demo里供大家学习。

