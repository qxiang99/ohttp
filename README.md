public class DataControl {


    private static DataControl dataControl;
    public OnDataControlListener listener;
    private static final String TAG = "DataControl";
    private boolean downSuccess = false;
    private Map<String, PlayData> PlayDataMap = new HashMap<>();
    private Map<String, PlayData> DownDataMap = new HashMap<>();
    private PlayData PlayDown;
    private long startDown = 0L;
    private long endDown = 0L;
    String PlayUUid = "DataControl";
    private QxDownload qxDownload;


    public static abstract class OnDataControlListener {
        public abstract void onNewPlayData(PlayAdvert playDataList);
    }

    public void SetDataControlListener(OnDataControlListener listener) {
        this.listener = listener;
        mHandler.sendEmptyMessageDelayed(1000, 1000);
        try {
            Calendar NowTime = SystemCalendarTime();
            PlayData data1 = DataPlay(UpTime(NowTime, NowTime.SECOND, 0), UpTime(NowTime, NowTime.DATE, +7), "", AppConstant.PLAYLISTUUID, SourceType.SourceTwo, DownType.DownloadThree);
            Play(null, data1);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        UpData();

    }

    public void getPlayData() {
        //进入数据处理
        ManageData();
    }


    public void DataEntry(AdvertListBean advertListBean) {
        try {
            //数据处理并存入数据库
            insertInformation(advertListBean);
            //进入数据处理
            ManageData();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


//********************************  数据处理  **************************************************

    //进入的数据处理
    private void ManageData() {
        try {
            //当前时间
            Calendar CurrentTime = SystemCalendarTime();
            //生成PlayData数据
            SetPlayData(CurrentTime);
            //
            Chronoscope();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    //进入的数据处理
    private void Chronoscope() throws Exception {
        //删除无用的数据
        DeleteData();

        PlayDataMap.clear();
        DownDataMap.clear();
        Calendar CurrentTime = SystemCalendarTime();
        List<PlayData> playDataList = DaoTool.SearchPlayDataAll();
        for (PlayData playData : playDataList) {
            Calendar StartTime = SystemCalendar(playData.getPlayStartTime());
            Calendar EndTime = SystemCalendar(playData.getPlayEndTime());
            //开始时间等于当前时间 || 当前时间在开始播放与结束播放时间之间并且离结束时间大于3秒 || 当前时间-开始时间小于3秒
            if (CurrentTime.compareTo(StartTime) == 0 || CurrentTime.after(StartTime) && CurrentTime.before(EndTime) && (EndTime.getTimeInMillis() - CurrentTime.getTimeInMillis() > 4000) || (StartTime.getTimeInMillis() - CurrentTime.getTimeInMillis() > 0 && StartTime.getTimeInMillis() - CurrentTime.getTimeInMillis() < 4000)) {
                PlayDataMap.put(UpTime(CurrentTime, CurrentTime.SECOND, +3), playData);
            } else if (CurrentTime.before(StartTime)) {
                //当前时间不在播放结束时间之后 && 当前时间不等于播放结束时间 && 开始播放时间大于当前时间3秒
                PlayDataMap.put(playData.getPlayStartTime(), playData);
                //如果是本地数据不用下载
                if (!playData.getPlayListUuid().equals(AppConstant.PLAYLISTUUID)) {
                    Calendar DownTime = SystemCalendar(playData.getDownloadTime());
                    //已经过了下载时间并且大于播放时间4s 设置下载时间
                    if (CurrentTime.after(DownTime)) {
                        if (StartTime.getTimeInMillis() - CurrentTime.getTimeInMillis() > 4000) {
                            DownDataMap.put(UpTime(CurrentTime, CurrentTime.SECOND, +3), playData);
                        }
                    } else {
                        //未到下载时间
                        DownDataMap.put(UpTime(DownTime, DownTime.SECOND, 0), playData);
                    }
                }
            }
        }

        for (String key : PlayDataMap.keySet()) {
            DebugLog.e(TAG, "........PlayDataMap..........." + key);
        }
        for (String key : DownDataMap.keySet()) {
            DebugLog.e(TAG, "........DownDataMap..........." + key);
        }
    }

    public Handler mHandler = new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            long now = SystemClock.uptimeMillis();
            long next = now + (1000 - now % 1000);
            mHandler.removeMessages(1000);
            mHandler.sendEmptyMessageAtTime(1000, next);

            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Calendar CurrentTime = SystemCalendarTime();
                        for (Iterator<Map.Entry<String, PlayData>> it = DownDataMap.entrySet().iterator(); it.hasNext(); ) {
                            Map.Entry<String, PlayData> item = it.next();
                            Calendar keyTime = SystemCalendar(item.getKey());
                            //开始前一秒，当前时间，开始之后一秒  三次机会，防止时间补偿失效跳秒的情况出现
                            if (CurrentTime.compareTo(keyTime) == 0 || CurrentTime.before(keyTime) && keyTime.getTimeInMillis() - CurrentTime.getTimeInMillis() < 1100 || CurrentTime.after(keyTime) && CurrentTime.getTimeInMillis() - keyTime.getTimeInMillis() < 2100) {
                                DebugLog.e(TAG, "....................当前是下载时间......................." + item.getKey());
                                final Information information = DaoTool.SearchInformation(item.getValue().getPlayListUuid());
                                it.remove();
                                Download(1, information, null);

                            }
                        }

                        for (Iterator<Map.Entry<String, PlayData>> it = PlayDataMap.entrySet().iterator(); it.hasNext(); ) {
                            Map.Entry<String, PlayData> item = it.next();
                            Calendar keyTime = SystemCalendar(item.getKey());
                            //开始前一秒，当前时间，开始之后一秒  三次机会，防止时间补偿失效跳秒的情况出现
                            if (CurrentTime.compareTo(keyTime) == 0 || CurrentTime.before(keyTime) && keyTime.getTimeInMillis() - CurrentTime.getTimeInMillis() < 1100 || CurrentTime.after(keyTime) && CurrentTime.getTimeInMillis() - keyTime.getTimeInMillis() < 3100) {
                                DebugLog.e(TAG, "....................当前是播放时间......................." + item.getKey());
                                String listuuid = item.getValue().getPlayListUuid();
                                Information information = DaoTool.SearchInformation(listuuid);
                                it.remove();
                                if (!listuuid.equals(PlayUUid)) {//防止正在播放的与要推给播放器的是一样的数据
                                    if (!item.getValue().getPlayListUuid().equals(AppConstant.PLAYLISTUUID)) {
                                        List<ImageModel> list = EffectiveData(item.getValue().getPlayListUuid());
                                        if (list.size() > 0) {
                                            DebugLog.e(TAG, "....................当前是播放时间...........数据不完整............");
                                            Download(2, information, item.getValue());
                                        } else {
                                            DebugLog.e(TAG, "....................当前是播放时间...........数据完整............");
                                            Play(information, item.getValue());
                                        }
                                    } else {
                                        Play(null, item.getValue());
                                    }
                                }
                            }
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();
            DebugLog.d(TAG, PlayDataMap.size() + "....................当前时间......................." + DateUtils.getInstance().getServerNowTime());
            return false;
        }
    });

    /**
     * 删除无用的数据
     */
    public void DeleteData()  {
        try {
            Calendar CurrentTime = SystemCalendarTime();
            List<Information> Newinformation = new ArrayList<>();
            List<Information> list = DaoTool.SearchInformation();
            Newinformation.addAll(list);
            List<String> UsefulList = new ArrayList<>();//不用的数据在新的数据里有引用
            for (Information information : Newinformation) {
                Calendar PlayPlayEndTime = SystemCalendar(information.getPlayEndTime());//广告数据的结束播放时间
                if (CurrentTime.after(PlayPlayEndTime)) {//当前时间在广告数据结束播放时间之后----数据无效
                    //删数据
                    DaoTool.DeteleInformation(information.getPlayListUuid());
                    DaoTool.DeteleAdvert(information.getPlayListUuid());
                } else {
                    List<Advert> advertList = DaoTool.SearchAdvert(information.getPlayListUuid());
                    for (final Advert advert : advertList) {
                        if (advert.getVedioFile() != null) {
                            UsefulList.add(advert.getVedioFile());
                        }
                        if (advert.getVedioCoverFile() != null) {
                            UsefulList.add(advert.getVedioCoverFile());
                        }
                        if (advert.getImageFile() != null) {
                            UsefulList.add(advert.getImageFile());
                        }
                    }
                }

            }
            List<String> uuidList = new ArrayList<>();//有用的uuid
            List<PlayData> playDataList = DaoTool.SearchPlayDataAll();
            for (PlayData playData : playDataList) {
                uuidList.add(playData.getPlayListUuid());
            }
            for (Information information : Newinformation) {
                if (!uuidList.contains(information.getPlayListUuid())) {
                    //删数据
                    DaoTool.DeteleInformation(information.getPlayListUuid());
                    DaoTool.DeteleAdvert(information.getPlayListUuid());
                }
            }

            List<Advert> advertList = DaoTool.SearchAdvert(AppConstant.PLAYLISTUUID);
            for (final Advert advert : advertList) {
                if (advert.getVedioFile() != null) {
                    UsefulList.add(advert.getVedioFile());
                }
                if (advert.getVedioCoverFile() != null) {
                    UsefulList.add(advert.getVedioCoverFile());
                }
                if (advert.getImageFile() != null) {
                    UsefulList.add(advert.getImageFile());
                }
            }

            List<String> Allfile = Utils.getFilesAllName(AppConstant.IMAGE_DATAIL);
            for (final String file : Allfile) {
                if (!UsefulList.contains(file)) {
                    new Thread(new Runnable() {
                        @Override
                        public void run() {
                            //删数据
                            Utils.delete(file);
                        }
                    }).start();
                }
            }
        } catch (ParseException e) {
            e.printStackTrace();
        }










       /* Calendar CurrentTime = SystemCalendarTime();
        List<PlayData> playDataList = DaoTool.SearchPlayDataAll();
        List<String> UsefulList = new ArrayList<>();//不用的数据
        List<String> uuidList = new ArrayList<>();//有用的uuid
        for (PlayData playData : playDataList) {
            Calendar PlayPlayEndTime = SystemCalendar(playData.getPlayEndTime());//广告数据的结束播放时间
            if (CurrentTime.before(PlayPlayEndTime)) {//当前时间在广告数据结束播放时间之后----数据无效
                uuidList.add(playData.getPlayListUuid());
                List<Advert> advertList = DaoTool.SearchAdvert(playData.getPlayListUuid());
                for (final Advert advert : advertList) {
                    if (advert.getVedioFile() != null) {
                        UsefulList.add(advert.getVedioFile());
                    }
                    if (advert.getVedioCoverFile() != null) {
                        UsefulList.add(advert.getVedioCoverFile());
                    }
                    if (advert.getImageFile() != null) {
                        UsefulList.add(advert.getImageFile());
                    }
                }
            }
        }

        List<Information> Newinformation = new ArrayList<>();
        List<Information> list = DaoTool.SearchInformation();
        Newinformation.addAll(list);
        for (Information information : Newinformation) {
            if (!uuidList.contains(information.getPlayListUuid())) {
                //删数据
                DaoTool.DeteleInformation(information.getPlayListUuid());
                DaoTool.DeteleAdvert(information.getPlayListUuid());
            }
        }

        List<Advert> advertList = DaoTool.SearchAdvert(AppConstant.PLAYLISTUUID);
        for (final Advert advert : advertList) {
            if (advert.getVedioFile() != null) {
                UsefulList.add(advert.getVedioFile());
            }
            if (advert.getVedioCoverFile() != null) {
                UsefulList.add(advert.getVedioCoverFile());
            }
            if (advert.getImageFile() != null) {
                UsefulList.add(advert.getImageFile());
            }
        }

        List<String> Allfile = Utils.getFilesAllName(AppConstant.IMAGE_DATAIL);
        for (final String file : Allfile) {
            if (!UsefulList.contains(file)) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        //删数据
                        Utils.delete(file);
                    }
                }).start();
            }
        }*/
    }

    /**
     * 执行播放回调
     */
    private void Play(Information information, PlayData playData) {
        //进行播放前进行数据校验
        PlayAdvert playAdvert = new PlayAdvert();
        //
        if (information != null) {
            information.setNowPlayStartTime(SystemTime());
            // DaoTool.insertInformation(information);

            playAdvert.setNowPlayStartTime(information.getNowPlayStartTime());
            playAdvert.setNowDownStartTime(information.getNowDownStartTime());
            playAdvert.setNowDownEndTime(information.getNowDownEndTime());
        } else {
            playAdvert.setNowPlayStartTime(SystemTime());
            playAdvert.setNowDownStartTime("0");
            playAdvert.setNowDownEndTime("0");
        }
        playAdvert.setPlayListUuid(playData.getPlayListUuid());
        playAdvert.setSourceType(playData.getSourceType());
        playAdvert.setDownType(playData.getDownType());
        playAdvert.setPlayStartTime(playData.getPlayStartTime());
        playAdvert.setPlayEndTime(playData.getPlayEndTime());
        List<Advert> PlayList = DaoTool.SearchAdvert(playData.getPlayListUuid());
        //读取所有playlist数据
        List<AdvertBean> PlayItemList = new ArrayList<>();
        for (Advert advert : PlayList) {
            AdvertBean advertBean = new AdvertBean();
            advertBean.setItemUuid(advert.getItemUuid());
            advertBean.setShowPosition(advert.getShowPosition());
            advertBean.setShowTime(advert.getShowTime() * 1000);
            advertBean.setVedioUuid(advert.getVedioUuid());
            advertBean.setVedioFile(advert.getVedioFile());
            advertBean.setVedioCoverFile(advert.getVedioCoverFile());
            advertBean.setImageUuid(advert.getImageUuid());
            advertBean.setImageFile(advert.getImageFile());
            advertBean.setOrderNum(advert.getOrderNum());
            PlayItemList.add(advertBean);
        }
        List<AdvertBean> NewAdvert = sortAdvert(PlayItemList);
        playAdvert.setPlayItem(NewAdvert);
        playAdvert.setNextPlayListUuid(playData.getNextPlayListUuid());
        PlayUUid = playAdvert.getPlayListUuid();
        listener.onNewPlayData(playAdvert);
    }


    public void Download(final int type, Information information, final PlayData playData) throws Exception {
        new Thread(new Runnable() {
            @Override
            public void run() {
                List<ImageModel> list = EffectiveData(information.getPlayListUuid());

                if (list.size() > 0) {
                    startDown = SystemTimeL();
                    PlayListLog(AppConstant.DOWNLOADSTART, information.getPlayListUuid());
                    information.setNowDownStartTime(SystemTime());
                    qxDownload = QxDownload.getInstance();
                    qxDownload.removeOnAllTask();
                    qxDownload.removeAll();
                    for (int i = 0; i < list.size(); i++) {
                        GetRequest<File> request = OkGo.<File>get(list.get(i).url);
                        qxDownload.request(list.get(i).url, request)
                                .priority(list.get(i).priority)
                                .extra1(list.get(i))
                                .save();
                    }
                    qxDownload.startAll();
                    qxDownload.addOnAllTaskEndListener(new Qxxecutor.OnAllTaskEndListener() {
                        @Override
                        public void onAllTaskEnd() {
                            qxDownload = null;
                            Log.e("onAllTaskEnd", "...........onAllTaskEnd........");
                            PlayListLog(AppConstant.DOWNLOADEND, information.getPlayListUuid());
                            information.setNowDownEndTime(String.valueOf(endDown));
                            DaoTool.insertInformation(information);
                            if (type == 2) {
                                Play(information, playData);
                            }
                        }
                    });
                } else {
                    if (type == 2) {
                        Play(information, playData);
                    }
                }
            }
        }).start();

    }

    private List<ImageModel> EffectiveData(String listuuid) {
        final List<ImageModel> list = new ArrayList<>();
        List<Advert> advertlist = DaoTool.SearchAdvert(listuuid);
        try {
            for (Advert advert : advertlist) {
                if (advert.getVedioDownloadUrl() != null && advert.getVedioSize() > 0) {
                    boolean boo = Utils.Compare(advert.getVedioSize(), advert.getVedioFile());
                    if (!boo) {
                        ImageModel imageModel2 = new ImageModel(advert.getVedioDownloadUrl());
                        list.add(imageModel2);
                    }
                }
                if (advert.getVedioCoverImgUrl() != null && advert.getCoverImgSize() > 0) {
                    ImageModel imageModel2 = new ImageModel(advert.getVedioCoverImgUrl());
                    list.add(imageModel2);
                }
                if (advert.getImageDownloadUrl() != null && advert.getImageSize() > 0) {
                    boolean boo = Utils.Compare(advert.getImageSize(), advert.getImageFile());
                    if (!boo) {
                        ImageModel imageModel3 = new ImageModel(advert.getImageDownloadUrl());
                        list.add(imageModel3);
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return list;
    }

    private void PlayListLog(String action, String playListUuid) {
        endDown = SystemTimeL();
        PlayListLogBean playListLogBean = new PlayListLogBean();
        playListLogBean.setFromAdSpotUuid(ControlManager.getInstance().adSpotUuid);
        playListLogBean.setSerialNumber(Utils.ANDROID_ID);
        playListLogBean.setTime(String.valueOf(endDown));
        playListLogBean.setListUuid(playListUuid);
        playListLogBean.setAction(action);
        if (action.equals(AppConstant.DOWNLOADEND)) {
            PlayListLogBean.ListInfoBean listInfoBean = new PlayListLogBean.ListInfoBean();
            listInfoBean.setStartTime(String.valueOf(startDown));
            listInfoBean.setEndTime(String.valueOf(endDown));
            listInfoBean.setSeconds(endDown - startDown);
            listInfoBean.setSize(0);
            listInfoBean.setStatus(0);
            playListLogBean.setListInfo(listInfoBean);
        }

        EventMsg eventMsg = new EventMsg();
        eventMsg.setCode(AppConstant.RS_PLAYLIST_LOG);
        eventMsg.setPlayListLogBean(playListLogBean);
        RxBus.getInstance().post(eventMsg);
    }

    private void SetPlayData(Calendar CurrentTime) throws Exception {
        //读取所有playlist数据
        List<Information> List = DaoTool.SearchInformation();
        List<Information> PlayList = sortInformation(List);
        List<PlayData> TimeList = new ArrayList<>();
        //获取所有播放时间
        if (PlayList != null && PlayList.size() > 0) {

            for (int i = 0; i < PlayList.size(); i++) {
                DataProcess(TimeList, PlayList.get(i));
            }
            //补充断档空隙
            List<PlayData> SupplementList = new ArrayList<>();
            try {
                for (int i = 0; i < TimeList.size(); i++) {
                    if (i == 0) {
                        // A 在PlayData之前，补充数据播放
                        Calendar NowTime = SystemCalendarTime();
                        Calendar StartTime = SystemCalendar(TimeList.get(i).getPlayStartTime());
                        if ((StartTime.getTimeInMillis() - NowTime.getTimeInMillis()) / 1000 > 1) {
                            PlayData data1 = DataPlay(UpTime(NowTime, NowTime.SECOND, 0), UpTime(StartTime, StartTime.SECOND, 0), "", AppConstant.PLAYLISTUUID, SourceType.SourceTwo, DownType.DownloadThree);
                            SupplementList.add(data1);
                        }
                    }
                    SupplementList.add(TimeList.get(i));
                    if (i + 1 < TimeList.size()) {
                        Calendar LastEndTime = SystemCalendar(TimeList.get(i).getPlayEndTime());
                        Calendar NextStartTime = SystemCalendar(TimeList.get(i + 1).getPlayStartTime());
                        if ((NextStartTime.getTimeInMillis() - LastEndTime.getTimeInMillis()) / 1000 > 1) {
                            PlayData playData = DataPlay(UpTime(LastEndTime, LastEndTime.SECOND, 0), UpTime(NextStartTime, NextStartTime.SECOND, 0), "", AppConstant.PLAYLISTUUID, SourceType.SourceTwo, DownType.DownloadThree);
                            SupplementList.add(playData);
                        }
                    } else {
                        Calendar AfterTime = SystemCalendar(TimeList.get(i).getPlayEndTime());
                        // B 在PlayData之后，补充数据播放，并拉取最新数据
                        PlayData playData = DataPlay(UpTime(AfterTime, AfterTime.SECOND, 0), UpTime(AfterTime, AfterTime.DATE, +7), "", AppConstant.PLAYLISTUUID, SourceType.SourceTwo, DownType.DownloadThree);
                        SupplementList.add(playData);

                    }
                }
            } catch (ParseException e) {
                e.printStackTrace();
            }
            TimeList.clear();
            TimeList.addAll(SupplementList);
        } else {
            PlayData playData = DataPlay(UpTime(CurrentTime, CurrentTime.SECOND, +2), UpTime(CurrentTime, CurrentTime.DATE, +7), "", AppConstant.PLAYLISTUUID, SourceType.SourceTwo, DownType.DownloadThree);
            TimeList.add(playData);
        }
        //添加下一条的播放uuid----上传日志需要
        List<PlayData> SupplementList = new ArrayList<>();
        for (int i = 0; i < TimeList.size(); i++) {
            if (TimeList.size() == 1) {
                PlayData playData = DataPlay(TimeList.get(i).getPlayStartTime(), TimeList.get(i).getPlayEndTime(), TimeList.get(i).getDownloadTime(), TimeList.get(i).getPlayListUuid(), TimeList.get(i).getSourceType(), TimeList.get(i).getDownType());
                playData.setNextPlayListUuid(TimeList.get(i).getPlayListUuid());
                SupplementList.add(playData);
            }
            if (i + 1 < TimeList.size()) {
                PlayData playData = DataPlay(TimeList.get(i).getPlayStartTime(), TimeList.get(i).getPlayEndTime(), TimeList.get(i).getDownloadTime(), TimeList.get(i).getPlayListUuid(), TimeList.get(i).getSourceType(), TimeList.get(i).getDownType());
                playData.setNextPlayListUuid(TimeList.get(i + 1).getPlayListUuid());
                SupplementList.add(playData);
            } else {
                if (TimeList.size() != 1) {
                    PlayData playData = DataPlay(TimeList.get(i).getPlayStartTime(), TimeList.get(i).getPlayEndTime(), TimeList.get(i).getDownloadTime(), TimeList.get(i).getPlayListUuid(), TimeList.get(i).getSourceType(), TimeList.get(i).getDownType());
                    playData.setNextPlayListUuid(TimeList.get(i).getPlayListUuid());
                    SupplementList.add(playData);
                }
            }
        }
        DaoTool.DeteleAllPlayData();
        for (PlayData playData : SupplementList) {
            DaoTool.insertPlayData(playData);
        }

    }


    private void DataProcess(List<PlayData> timeList, Information information) throws Exception {
        if (timeList.size() == 0) {
            PlayData playData = DataPlay(information.getPlayStartTime(), information.getPlayEndTime(), information.getDownloadTime(), information.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
            timeList.add(playData);
        } else {
            List<PlayData> NewTimeList = new ArrayList<>();
            //数据生成
            for (PlayData playData : timeList) {
                SetTime(NewTimeList, playData, information);
            }
            //数据筛选
            if (timeList.size() > 1) {
                ModifyTime(NewTimeList);
            }
            timeList.clear();
            timeList.addAll(NewTimeList);
        }
    }

    private void SetTime(List<PlayData> newTimeList, PlayData playData, Information information) throws Exception {

        Calendar TimeBeanStartTime = SystemCalendar(playData.getPlayStartTime());
        Calendar TimeBeanEndTime = SystemCalendar(playData.getPlayEndTime());
        Calendar InformationStartTime = SystemCalendar(information.getPlayStartTime());
        Calendar InformationEndTime = SystemCalendar(information.getPlayEndTime());

        if (TimeBeanStartTime.before(TimeBeanEndTime) && InformationStartTime.before(InformationEndTime)) {
            if (TimeBeanEndTime.before(InformationStartTime) || TimeBeanEndTime.compareTo(InformationStartTime) == 0) {
                //AB CD  ---A<B<C<D    A<B=C<D    A<C=B<D
                PlayData data1 = DataPlay(playData.getPlayStartTime(), playData.getPlayEndTime(), playData.getDownloadTime(), playData.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data1);
                PlayData data2 = DataPlay(information.getPlayStartTime(), information.getPlayEndTime(), information.getDownloadTime(), information.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data2);

            } else if (TimeBeanStartTime.before(InformationStartTime) && TimeBeanStartTime.before(InformationEndTime) && TimeBeanEndTime.after(InformationStartTime) && TimeBeanEndTime.after(InformationEndTime)) {
                //AC CD DB--A<C<D<B
                PlayData data1 = DataPlay(playData.getPlayStartTime(), UpTime(InformationStartTime, InformationStartTime.SECOND, 0), playData.getDownloadTime(), playData.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data1);
                PlayData data2 = DataPlay(information.getPlayStartTime(), information.getPlayEndTime(), information.getDownloadTime(), information.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data2);
               /* PlayData data3 = DataPlay(UpTime(InformationEndTime, InformationEndTime.SECOND, 0), playData.getPlayEndTime(), playData.getDownloadTime(), playData.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data3);*/

            } else if (TimeBeanStartTime.before(InformationStartTime) && TimeBeanEndTime.after(InformationStartTime) && (TimeBeanEndTime.before(InformationEndTime) || TimeBeanEndTime.compareTo(InformationEndTime) == 0)) {
                //AC CD  ---A<C<B<D    A<C<B=D    A<C<D=B
                PlayData data1 = DataPlay(playData.getPlayStartTime(), UpTime(InformationStartTime, InformationStartTime.SECOND, 0), playData.getDownloadTime(), playData.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data1);
                PlayData data2 = DataPlay(information.getPlayStartTime(), information.getPlayEndTime(), information.getDownloadTime(), information.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data2);

            } else if (TimeBeanEndTime.after(InformationStartTime) && (TimeBeanStartTime.compareTo(InformationEndTime) == 0 || InformationEndTime.before(TimeBeanStartTime))) {
                //CD AB  ---C<D<A<B    C<D=A<B     C<A=D<B
                PlayData data1 = DataPlay(information.getPlayStartTime(), information.getPlayEndTime(), information.getDownloadTime(), information.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data1);
                PlayData data2 = DataPlay(playData.getPlayStartTime(), playData.getPlayEndTime(), playData.getDownloadTime(), playData.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data2);

            } else if (TimeBeanEndTime.after(InformationStartTime) && TimeBeanStartTime.before(InformationEndTime) && InformationEndTime.before(TimeBeanEndTime) && (TimeBeanStartTime.compareTo(InformationStartTime) == 0 || InformationStartTime.before(TimeBeanStartTime))) {
                //CD DB  ---C<A<D<B    C=A<D<B    A=C<D<B
                PlayData data1 = DataPlay(information.getPlayStartTime(), information.getPlayEndTime(), information.getDownloadTime(), information.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data1);
                PlayData data2 = DataPlay(UpTime(InformationEndTime, InformationEndTime.SECOND, 0), playData.getPlayEndTime(), playData.getDownloadTime(), playData.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data2);

            } else if (TimeBeanStartTime.before(InformationEndTime) && InformationStartTime.before(TimeBeanEndTime) && (TimeBeanStartTime.compareTo(InformationStartTime) == 0 && (TimeBeanEndTime.before(InformationEndTime) || TimeBeanEndTime.compareTo(InformationEndTime) == 0) || InformationStartTime.before(TimeBeanStartTime) && (TimeBeanEndTime.before(InformationEndTime) || TimeBeanEndTime.compareTo(InformationEndTime) == 0))) {
                //   CD  ---A=C<B<D    A=C<B=D    A=C<D=B    C<A<D=B   C<A<B<D
                //          C=A<B<D    C=A<B=D    C=A<D=B    C<A<B=D
                PlayData data2 = DataPlay(information.getPlayStartTime(), information.getPlayEndTime(), information.getDownloadTime(), information.getPlayListUuid(), SourceType.SourceOne, DownType.DownloadOne);
                newTimeList.add(data2);

            } else {
                //时间错误
                System.out.println("..........第二步错误");
            }
        } else {
            //时间错误
            System.out.println("..........第一步错误");
        }

    }

    //排序---现去重---在按时间排序
    private void ModifyTime(List<PlayData> newTimeList) {
        for (int i = 0; i < newTimeList.size() - 1; i++) {
            for (int j = newTimeList.size() - 1; j > i; j--) {
                if (newTimeList.get(j).getPlayListUuid() == newTimeList.get(i).getPlayListUuid() && newTimeList.get(j).getPlayStartTime() == newTimeList.get(i).getPlayStartTime() && newTimeList.get(j).getPlayEndTime() == newTimeList.get(i).getPlayEndTime()) {
                    newTimeList.remove(j);
                }
            }
        }
        SortList(newTimeList);

    }

    private List<PlayData> SortList(List<PlayData> newTimeList) {
        Collections.sort(newTimeList, new Comparator<PlayData>() {
            @Override
            public int compare(PlayData o1, PlayData o2) {
                if (o1.getPlayStartTime().compareTo(o2.getPlayStartTime()) > 0) {
                    return 1;
                } else {
                    return -1;
                }
            }
        });
        return newTimeList;
    }

    private List<AdvertBean> sortAdvert(List<AdvertBean> newTimeList) {
        Collections.sort(newTimeList, new Comparator<AdvertBean>() {
            @Override
            public int compare(AdvertBean o1, AdvertBean o2) {
                if (o1.getOrderNum() > o2.getOrderNum()) {
                    return 1;
                } else {
                    return -1;
                }
            }
        });
        return newTimeList;
    }

    private List<Information> sortInformation(List<Information> newTimeList) {
        List<Information> list = new ArrayList<>();
        list.addAll(newTimeList);
        Collections.sort(list, new Comparator<Information>() {
            @Override
            public int compare(Information o1, Information o2) {
                if (o1.getTimestramp() > o2.getTimestramp()) {
                    return 1;
                } else {
                    return -1;
                }
            }
        });
        return list;
    }

    private Information insertInformation(AdvertListBean advertListBean) {
        Information information = new Information();
        information.setPlayListUuid(advertListBean.getPlayListUuid());
        information.setPlayStartTime(advertListBean.getPlayStartTime());
        information.setPlayEndTime(advertListBean.getPlayEndTime());
        information.setDownloadTime(advertListBean.getDownloadTime());
        information.setDownloadType(DownType.DownloadOne);
        information.setResourceTotalSize(advertListBean.getResourceTotalSize());
        information.setTimestramp(advertListBean.getTimestramp());
        information.setCreatTime(DateUtils.getInstance().getServerNowTime());
        DaoTool.insertInformation(information);
        DaoTool.DeteleAdvert(information.getPlayListUuid());
        for (AdvertListBean.PlayListBean playListBean : advertListBean.getPlayList()) {
            Advert advert = new Advert();
            advert.setUuid(playListBean.getItemUuid() + information.getPlayListUuid());
            advert.setInformationuuid(information.getPlayListUuid());
            advert.setItemUuid(playListBean.getItemUuid());
            advert.setShowPosition(playListBean.getShowPosition());
            advert.setShowTime(playListBean.getShowTime());
            advert.setOrderNum(playListBean.getOrderNum());
            if (playListBean.getVedio() != null) {
                advert.setVedioUuid(playListBean.getVedio().getVedioUuid());
                advert.setVedioSize(playListBean.getVedio().getVedioSize());
                advert.setVedioDownloadUrl(playListBean.getVedio().getDownloadUrl());
                advert.setVedioCoverImgUrl(playListBean.getVedio().getCoverImgUrl());
                advert.setVedioFile(AppConstant.IMAGE_DATAIL + "/" + playListBean.getVedio().getFileName());
                advert.setCoverImgSize(playListBean.getVedio().getCoverImgSize());
                advert.setVedioCoverFile(AppConstant.IMAGE_DATAIL + "/" + playListBean.getVedio().getCoverImgFileName());
            }
            if (playListBean.getImage() != null) {
                advert.setImageUuid(playListBean.getImage().getImageUuid());
                advert.setImageSize(playListBean.getImage().getImageSize());
                advert.setImageDownloadUrl(playListBean.getImage().getDownloadUrl());
                advert.setImageFile(AppConstant.IMAGE_DATAIL + "/" + playListBean.getImage().getFileName());
            }
            DaoTool.insertAdvert(advert);
        }
        Information infor = DaoTool.SearchInformation(advertListBean.getPlayListUuid());
        return infor;
    }

    private PlayData DataPlay(String startTime, String endTime, String downloadTime, String listuuid, int soucretype, int downtype) {
        PlayData playData = new PlayData();
        playData.setPlayStartTime(startTime);
        playData.setPlayEndTime(endTime);
        playData.setDownloadTime(downloadTime);
        playData.setPlayListUuid(listuuid);
        playData.setSourceType(soucretype);
        playData.setDownType(downtype);
        return playData;
    }

    private String UpTime(Calendar currentTime, int field, int amount) {
        SimpleDateFormat SimpleFormatTime = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        currentTime.add(field, amount);
        return SimpleFormatTime.format(currentTime.getTime());
    }

    private Calendar SystemCalendar(String TIME) throws ParseException {
        SimpleDateFormat SimpleFormatTime = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(SimpleFormatTime.parse(TIME));
        return calendar;
    }

    private String SystemString() {
        SimpleDateFormat SimpleFormatTime = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return SimpleFormatTime.format(new Date(DateUtils.getInstance().getServerElapsedTime()));
    }

    private String SystemTime() {
        return String.valueOf(SystemTimeL());
    }

    private long SystemTimeL() {
        return DateUtils.getInstance().getServerElapsedTimeL();
    }

    private Calendar SystemCalendarTime() throws ParseException {//DateUtils.getInstance().getServerNowTime()
        SimpleDateFormat SimpleFormatTime = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(SimpleFormatTime.parse(DateUtils.getInstance().getServerNowTime()));
        return calendar;
    }

    //拉取服务器最新的playlist数据
    private void UpData() {
        EventMsg eventMsg = new EventMsg();
        eventMsg.setCode(AppConstant.RS_LAYOUT_DATA);
        RxBus.getInstance().post(eventMsg);
    }


    public void Stop() {
        mHandler.removeMessages(1000);
    }
}

