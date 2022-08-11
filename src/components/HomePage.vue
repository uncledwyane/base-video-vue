<template>
    <div class="homepage">
        <div :class="'notify notify-' + notify.type" v-show="notify.show">{{ notify.content }}</div>
        <div class="remote-user-wrap">
            <video-box type='remote' :user="remoteUser" ref="remote" v-if="remoteUser != null"></video-box>
        </div>
        <video-box type='local' :user="localUser" ref="local" v-on:stream-switch="handleStreamSwitch" v-if="localUser != null"></video-box>
        <span class="company-tag">base-video-vue by 3tee    当前房间号：{{ roomId }}   </span>
        <div class="join-layer" v-show="!joinSuccess">
            <div class="join-box">
                <div class="prepare-join-info">
                    <input type="text" class="join-info-room-id" v-model="roomId" placeholder="房间号">
                    <span class="join-info-room-get-btn" @click="getRoomByCreateRoom">生成</span>
                </div>
                <button class="join-btn" @click="joinRoom"><i class="loading-icon" v-show="joinLoading"></i>加入会议</button>
            </div>
        </div>
    </div>
</template>

<script>
import VideoBox from './VideoBox.vue'
import config from '../config.js'
import log from '../utils/util.js'
export default {
    data() {
        return {
            restApi: new RestApi(config.serverURI),
            token: '',
            notify: {
                show: false,
                content: '',
                type: ''
            },
            room: null,
            roomId: '',
            userId: '',
            userName: '',
            remoteUser: null,
            localUser: null,
            browserInfo: null,
            notifyTimeout: null,
            joinSuccess: false,
            joinLoading: false
        }
    },
    components: {
        VideoBox
    },
    mounted() {
        this.init()
    },
    methods: {
        init() {
            // 设置日志级别为debug
            avdEngine.setLog(Appender.browserConsole, LogLevel.debug);
            // 初始化音视频设备
            avdEngine.checkDevice().then(() => {
                this.setNotify({content: '设备初始化开始'})
                avdEngine.initDevice().then(state => {
                    state === 1 ? console.log('+++initDevices', state) : ''
                    // 获取token
                    // 获取浏览器信息
                    this.browserInfo = avdEngine.getBrowserDetect().browser
                    
                    this.userId = this.browserInfo.name + Math.round(Math.random() * 10000000000);
                    this.userName = this.browserInfo.name + ' ' + this.browserInfo.fullVersion;
                    this.setNotify({type: 'success', content: '设备初始化完成'})
                    this.getToken().then(() => {
                        // this.getRoomByCreateRoom()
                    }).catch(err => {
                        this.setNotify({
                            type: 'error',
                            content: '获取token失败了'
                        })
                    })
                })
            })
        },
        getToken() {
            return new Promise((resolve, reject) => {
                this.restApi.getAccessToken(config.accessKey, config.secretKey).then(res => {
                    this.token = res.token
                    resolve()
                })
            })
        },
        joinRoom() {
            const self = this;

            if(self.roomId === ''){
                this.setNotify({
                    type: 'warning',
                    content: '必须要房间号'
                })
                return
            }

            // 初始化
            self.joinLoading = true;
            avdEngine.init(config.serverURI, self.token).then(self.engineInitSuccess).otherwise(self.engineInitFaild);
        },
        getRoomByCreateRoom() {
            const self = this;
            let topic = 'testingMeeting';
            // 调接口创建房间
            self.restApi.createRoom(self.token, topic, self.userId).then(function(res) {
                self.roomId = res.room_id
            })
        },
        engineInitSuccess() {
            const self = this;
            this.setNotify({type: 'success', content: '引擎初始化完成'})
            self.room = avdEngine.obtainRoom(self.roomId);
            self.room.join(self.userId, self.userName, '', '').then(self.joinRoomSuccess).otherwise(self.joinRoomFaild)
        },
        engineInitFaild() {
            // 将初始化失败显示到通知栏内
        },
        joinRoomSuccess() {
            console.log('+++joinRoomSuccess')

            const self = this;



            self.registerRoomCallback();

            //加会登陆前，会议中已经发布的视频资源,采取订阅处理
            self.onPublishCameraNotify(room.pubVideos);

            // 处理房间内的参会者，给每个参会者添加回调事件
            self.participantsHandle(room.getParticipants());

            self.initClientUserFromSDKUser('local')

            self.setNotify({type: 'success', content: '加入会议成功'})

            self.switchVideo('open');
            // self.switchAudio('open');
            self.joinSuccess = true;
            this.joinLoading = false;
        },
        /**
         * 从sdk用户初始化到当前客户端用户
         * @param {string} type 类型 local | remote
         * @param {object} remoteUser 非本端用户
         */
        initClientUserFromSDKUser(type, remoteUser) {
            let user = {
                audio: type === 'local' ? room.selfUser.audio ? room.selfUser.audio : null : remoteUser.audio ? remoteUser.audio : null,
                video: type === 'local' ? room.selfUser.videos.length > 0 ? room.selfUser.videos[0] : null : remoteUser.videos.length > 0 ? remoteUser.videos[0] : null,
                id: type === 'local' ? room.selfUser.id : remoteUser.id,
                name: type === 'local' ? room.selfUser.name : remoteUser.name
            }

            type === 'local' ? this.localUser = user : this.remoteUser = user

            console.log('+++initClientUserFromSDKUser: ', user)
        }, 
        joinRoomFaild(err) {
            console.log('+++joinRoomFaild, err: ', err)
            this.setNotify({type: 'error', content: '加入会议失败，原因：' + err.message})
        },
        switchVideo(type) {
            log('switch video, type: ' + type)
            const self = this;
            const localUserDefaultCamera = self.localUser.video

            // 关闭视频
            if(type === 'close'){
                localUserDefaultCamera.unpublish()
                self.$refs['local'].renderStream('video', null)
            }

            // 打开视频
            if(type === 'open'){
                localUserDefaultCamera.publish().then(() => {
                    self.$refs['local'].renderStream('video', localUserDefaultCamera.getNewStreamByTrack())
                })
            }
        },
        /**
         * 根据类型操作音频
         * @param {string} type - 类型
         */
        switchAudio(type) {
            log('switch audio, type: ' + type)
            const self = this;
            const localUserDefaultAudio = self.localUser.audio
            // 关闭音频
            if(type === 'close' && (localUserDefaultAudio.status == StreamStatus.publish || localUserDefaultAudio.status == StreamStatus.muted)){
                localUserDefaultAudio.closeMicrophone()
                log('switch audio, close')
                return
            }

            // 打开音频
            if(type === 'open' && localUserDefaultAudio.status == StreamStatus.init){
                localUserDefaultAudio.openMicrophone()
                log('switch audio, open')
                return
            }

            // 静音
            if(type === 'mute' && (localUserDefaultAudio.status != StreamStatus.init || localUserDefaultAudio.status != StreamStatus.muted)){
                localUserDefaultAudio.muteMicrophone()
                log('switch audio, mute')
                return
            } 
            // 取消静音
            if(type === 'unmute' && localUserDefaultAudio.status != StreamStatus.init){
                localUserDefaultAudio.unmuteMicrophone()
                log('switch audio, unmute')
                return
            } 
        },
        registerRoomCallback() {
            const self = this;
            self.room.addCallback(RoomCallback.connection_status, self.onConnectionStatus);
            self.room.addCallback(RoomCallback.user_join_notify, self.onUserJoinNotify);
            self.room.addCallback(RoomCallback.user_leave_notify, self.onUserLeaveNotify);
        },
        /**
         * @desc  用户回调事件
         * @param {Object} participants - 用户集数组
         */
        participantsHandle(participants) {
            const self = this;
            participants.forEach(function (user) {
                user.addCallback(UserCallback.camera_status_notify, self.onCameraStatusNotify);
                user.addCallback(UserCallback.microphone_status_notify, self.onMicrophoneStatusNotify);

                user.addCallback(UserCallback.publish_camera_notify, self.onPublishCameraNotify);
                user.addCallback(UserCallback.unpublish_camera_notify, self.onUnpublishCameraNotify);
                user.addCallback(UserCallback.subscrible_camera_result, self.onSubscribleCameraResult);
                user.addCallback(UserCallback.unsubscrible_camera_result, self.onUnsubscribleCameraResult);

                user.addCallback(UserCallback.subscrible_microphone_result, self.onSubscribleMicrophoneResult);
                user.addCallback(UserCallback.unsubscrible_microphone_result, self.onUnsubscribleMicrophoneResult);

                if(user.id !== room.selfUser.id){
                    self.initClientUserFromSDKUser('remote', user)
                }
            })
        },
        onConnectionStatus(status) {
            const self = this;
            if(status == ConnectionStatus.connecting) {
                self.setNotify({
                    type: 'error',
                    content: '网络故障,正在与服务器重连中...'
                })
            } else if(status == ConnectionStatus.connected) {
                //连接成功
                self.setNotify({
                    type: 'success',
                    content: '连接成功'
                })
            } else if(status == ConnectionStatus.reJoinConnected) {
                //重新加会成功
                self.setNotify({
                    type: 'success',
                    content: '重新加会成功'
                })
            } else if(status == ConnectionStatus.reconnected) {
                //重连接成功
                self.setNotify({
                    type: 'success',
                    content: '重新连接成功'
                })
            } else if(status == ConnectionStatus.connectFailed) {
                self.setNotify({
                    type: 'warning',
                    content: '网络故障,与服务器重连超时，正在重新加会操作中...'
                })
                //应用层清场
                if(room.connectionInfoCollector) {
                    room.connectionInfoCollector.stop(); //停止收集网络情况
                }
            } else if(status == ConnectionStatus.reJoinRoomTimeOut) {
                self.setNotify({
                    type: 'error',
                    content: '网络故障,重新加会中...'
                })
                /**
                 * @desc 多次后重新加会超时后，开启持续永久重新加会
                 */
                self.room.continuousReJoin();
            }
        },
        /**
         * @desc 参会者加会回调
         * @param {Object} users － 参会者数组
         */
        onUserJoinNotify(users) {
            const self = this;
            console.log("+++onUserJoinNotify(),users:", room.participants.length);
            if(room.participants.length <= 2){
                self.participantsHandle(users);
            }else{
                self.setNotify({
                    type: 'warning',
                    content: '当前demo仅允许两人参会，多余参会者不会处理'
                })
            }
        },
        /**
         * @desc 参会者退会回调
         * @param {int} opt - 退会类型
         * @param {int} reason  - 退会原因
         * @param {Object} user - 退会用户
         */
        onUserLeaveNotify(opt, reason, user) {
            const self = this;
            //UDP媒体通道建立超时807错误
            if(reason == 807) {
                if(user.id == room.selfUser.id){
                    self.setNotify({
                        type: 'warning',
                        content: "本端 807错误,UDP不通或UDP媒体通道建立超时!"
                    })
                }else {
                    self.setNotify({
                        type: 'warning',
                        content: user.name +" 807错误,UDP不通或UDP媒体通道建立超时!"
                    })
                }
                return;
            }

            if(user.id == room.selfUser.id){
                self.localUser = null
            }else{
                self.remoteUser = null
            }
        },
        /**
         * 摄像头状态更新
         * @param {Object} status － 状态
         * @param {Object} cameraId － 摄像头设备Id
         * @param {Object} cameraName－ 摄像头设备名称
         * @param {Object} userId－ 摄像头设备所属者ＩＤ
         */
        onCameraStatusNotify(status, cameraId, cameraName, userId) {
            
        },
        /**
         * 麦克风状态更新
         * @param {Object} status － 状态
         * @param {Object} microphoneId － 麦克风设备Id 
         * @param {Object} microphoneName － 麦克风设备名称
         * @param {Object} userId － 麦克风设备所属者ＩＤ
         */
        onMicrophoneStatusNotify(status, microphoneId, microphoneName, userId) {
            
        },
        //订阅未订阅的视频
        onPublishCameraNotify(videos) {
            videos.forEach(function (video) {
                //只订阅未订阅过的视频
                var subVideoIdsLen = room.selfUser.subVideoIds.length;
                if (subVideoIdsLen > 0) {
                    var isSub = false;
                    for (var i = 0; i < subVideoIdsLen; i++) {
                        var videoId = room.selfUser.subVideoIds[i];
                        if (video.id == videoId) {
                            isSub = true;
                            break;
                        }
                    }
                    if (!isSub) {
                        video.subscrible();
                    }
                } else {
                    video.subscrible();
                }
            });
        },
        /**
         * 房间中发布的视频回调
         * @param {Object} video-发布视频集
         */
        onUnpublishCameraNotify(video) {
            video.unsubscrible();
        },
        /**
         * 订阅远端视频流反馈
         * @param {Object} stream － 远端视频流
         * @param {Object} userId － 所属用户ＩＤ
         * @param {Object} userName－ 所属用户名称
         * @param {Object} cameraId－ 摄像头设备ＩＤ
         */
        onSubscribleCameraResult(stream, userId, userName, cameraId) {
            console.log('+++onSubscribleCameraResult, userId: ', userId, ',cameraId: ', cameraId)
            if (userId !== room.selfUser.id) {
                const user = room.getUser(userId)
                const video = user.getVideo(cameraId)
                this.remoteUser.video = video ? video : null
                this.$refs['remote'].renderStream('video', stream)
            }
        },
        /**
         * 取消订阅远端视频流反馈
         * @param {Object} userId－ 所属用户ＩＤ
         * @param {Object} userName－所属用户名称
         * @param {Object} cameraId－摄像头设备ＩＤ
         */
        onUnsubscribleCameraResult(userId, userName, cameraId) {
            console.log('+++onUnsubscribleCameraResult, userId: ', userId, ',cameraId: ', cameraId)
            if (userId !== room.selfUser.id) {
                const user = room.getUser(userId)
                const video = user.getVideo(cameraId)
                this.remoteUser.video = video ? video : null
                this.$refs['remote'].renderStream('video', null)
            }
        },
        /**
         * 订阅远端音频流反馈
         * @param {Object} stream－ 远端音频流
         * @param {Object} userId－ 所属用户ＩＤ
         * @param {Object} userName－所属用户名称
         */
        onSubscribleMicrophoneResult(stream, userId, userName) {
            console.log('+++onSubscribleMicrophoneResult, userId: ', userId)
            if (userId !== room.selfUser.id) {
                const user = room.getUser(userId)
                this.remoteUser.audio = user.audio ? user.audio : null
                this.$refs['remote'].renderStream('audio', stream)
            }
        },
        /**
         * 取消订阅远端音频流反馈
         * @param {Object} userId－ 所属用户ＩＤ
         * @param {Object} userName－所属用户名称
         */
        onUnsubscribleMicrophoneResult(userId, userName) {
            console.log('+++onUnsubscribleMicrophoneResult, userId: ', userId)
            if (userId !== room.selfUser.id) {
                const user = room.getUser(userId)
                this.remoteUser.audio = user.audio ? user.audio : null
                this.$refs['remote'].renderStream('audio', null)
            }
        },
        setNotify(notifyObject) {
            const self = this;
            const {type, content} = notifyObject;

            if(self.notifyTimeout){
                clearTimeout(self.notifyTimeout)
                self.notifyTimeout = null
            }

            self.notify = {
                show: true,
                type,
                content
            }

            self.notifyTimeout = setTimeout(() => {
                self.notify = {
                    show: false,
                    type: '',
                    content: ''
                }
            }, 5000)
        },
        handleStreamSwitch(switchObj) {
            console.log('handleStreamSwitch: ', switchObj)
            const {sourceType, switchType} = switchObj
            if(sourceType === 'video'){
                this.switchVideo(switchType)
            }

            if(sourceType === 'audio'){
                this.switchAudio(switchType)
            }
        }
    }
}
</script>

<style scoped>
    .homepage{
        width: 100%;
        height: 100%;
        position: relative;
    }
    .remote-user-wrap{
        width: 700px;
        height: 500px;
        position: relative;
        top: 50%;
        left: 50%;
        transform: translateX(-50%) translateY(-50%);
    }
    .notify{
        width: 100%;
        height: 40px;
        line-height: 40px;
        box-sizing: border-box;
        padding: 0 10px;
        text-align: center;
        position: absolute;
        top: 0;
        left: 0;
        background: rgba(36, 59, 85,.5);
        color: #ffffff;
        backdrop-filter: blur(5px);
        z-index: 1000;
    }
    .notify-warning{
        background: rgba(255, 230, 0, 0.3);
        backdrop-filter: blur(5px);
    }
    .notify-error{
        background: rgba(250, 0, 0, 0.3);
        backdrop-filter: blur(5px);
    }
    .notify-success{
        background: rgba(30, 255, 0, 0.3);
        backdrop-filter: blur(5px);
    }
    .company-tag{
        display: block;
        position: absolute;
        left: 10px;
        bottom: 10px;
        color: rgb(131, 131, 131);
    }
    .join-layer{
        display: flex;
        align-items: center;
        justify-content: center;
        width: 100%;
        height: 100%;
        position: absolute;
        top: 0;
        left: 0;
        background: rgba(0,0,0,.3);
        backdrop-filter: blur(5px);
    }
    .join-btn{
        font-size: 1.25rem;
        padding: 10px 30px;
        border-radius: 40px;
        color: #ffffff;
        outline: none;
        border: none;
        width: 200px;
        background: rgb(61, 99, 143);
        cursor: pointer;
        height: 50px;
        line-height: 50px;
        transition: all ease .3s;
        display: flex;
        align-items: center;
        justify-content: center;
        user-select: none;
    }
    .join-btn:hover{
        background: rgba(36, 59, 85, 1);
    }
    .loading-icon{
        display: inline-block;
        width: 30px;
        height: 30px;
        background: url('../assets/images/loading.svg') no-repeat center center;
        background-size: contain;
        position: relative;
        left: -10px;
        animation: loading infinite linear 1s;
    }
    .join-box{
        width: 400px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
    .prepare-join-info{
        position: relative;
        width: 100%;
        height: 80px;
    }
    .join-info-room-id{
        width: 100%;
        height: 50px;
        outline: none;
        border: none;
        display: block;
        position: absolute;
        left: -3px;
        top: 0;
        background: rgb(36, 59, 85);
        color: #ffffff;
        border-radius: 10px;
        padding: 10px 20px;
        font-size: 1.25rem;
        box-sizing: border-box;
    }

    .join-info-room-get-btn{
        display: inline-block;
        height: 40px;
        color: #ffffff;
        z-index: 100;
        position: absolute;
        background: #162333;
        box-sizing: border-box;
        text-align: center;
        line-height: 40px;
        width: 50px;
        right: 10px;
        top: 5px;
        border-radius: 7px;
        cursor: pointer;
        transition: all ease .3s;
        user-select: none;
    }

    .join-info-room-get-btn:hover{
        background: #111a25;
    }

    @keyframes loading {
        from {
            transform: rotate(0deg);
        }

        to {
            transform: rotate(360deg);
        }
    }
</style>