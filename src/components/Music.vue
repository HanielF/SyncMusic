<template>
  <div>
    <aplayer
      :music="music" preload="auto" :list="musiclist" ref="musicplayer"  :class="{hidden_controller:!isRoomOwner}" v-if="isGetDataComplete" />
    <div id="HelloDiv"  v-if="firstopen">
      <div id="agereeButton" @click="agree">
        <span>开始使用</span>
      </div>
    </div>
    <audio id="emitaudio" src="static/audio/emit.mp3"></audio>
  </div>
</template>

<script>
import Aplayer from "vue-aplayer";
Aplayer.disableVersionBadge = true;
import axios from "axios";
import swal from "sweetalert2";
export default {
  components: {
    Aplayer
  },
  methods: {
    //设置监听器
    setEventListener: function() {
      var self = this;
      var audio = document.getElementsByTagName("audio")[0];
      //seeked在跳跃操作完成时触发。
      audio.addEventListener("seekd", self.seekfunction);
      //pause在暂停的时候触发
      audio.addEventListener("pause", self.sendJson);
      //canplay在媒体数据已经有足够的数据（至少播放数帧）可供播放时触发
      audio.addEventListener("canplay", self.setplay);
      //play在媒体回放被暂停后再次开始时触发。
      audio.addEventListener("play", self.sendJson);
      audio.addEventListener("ended", self.enddealy);
    },
    //取消设置监听器
    cancleEventListener: function() {
      var self = this;
      var audio = document.getElementsByTagName("audio")[0];
      //seeked在跳跃操作完成时触发。
      audio.removeEventListener("seekd", self.sendJson);
      //pause在暂停的时候触发
      audio.removeEventListener("pause", self.sendJson);
      //canplay在媒体数据已经有足够的数据（至少播放数帧）可供播放时触发
      audio.removeEventListener("canplay", self.setplay);
      //play在媒体回放被暂停后再次开始时触发。
      audio.removeEventListener("play", self.sendJson);
      //pause在停止的时候触发
      audio.addEventListener("ended", self.enddealy);
    },
    //初始化数据
    initData: function() {
      var self = this;
      axios.get(this.configurl, {}).then(function(response) {
        //lylares 请求方式
        var data = response.data;
        data = eval("(" + data.params.data + ")").sync;
        self.songid = data.listid;
        if (data.listid != undefined && data.listid != "") {
          $.getJSON(
            "https://163.fczbl.vip/playlist/detail",
            { id: data.listid },
            function(data, textStatus, jqXHR) {
              // var data = response.data;
              console.log(data)
              var temp = 0;
              if (data.code == "200") {
                //进行数据处理
                //获取到播放列表
                // var playlist = data.playlist;
                var tracks = data.playlist.tracks;
                var arr = new Array();
                console.log(tracks);
                tracks.forEach(element => {
                  //歌曲标题
                  var name = element.name;
                  //歌曲id
                  var id = element.id;
                  //作家
                  var artist = "未知";
                  if (
                    element.ar[0] != null &&
                    element.ar[0].name != null
                  ) {
                    artist = element.ar[0].name;
                  }
                  //封面
                  var pic_img = "";
                  if (element.al != null && element.al.picUrl != null) {
                    pic_img = element.al.picUrl;
                  }
                  //播放地址
                  // https://api.paugram.com/netease/?play=true&id=517567145
                  // var url =
                  //   "https://api.imjad.cn/cloudmusic/?type=song&raw=true&id=" +
                  //   id;
                  var url =
                    "https://api.paugram.com/netease/?play=true&id=" + id;
                  var copyright = element.copyright;
                  //排除所有没有版权的歌
                  // if (copyright == 1) {
                  arr[temp] = {
                    title: name,
                    artist: artist,
                    src: url,
                    pic: pic_img
                  };
                  temp++;
                  // }
                });
                //设置当前的listid
                self.music = arr[0];
                self.musiclist = arr;
                self.isGetDataComplete = true;
                //隐藏开屏
                //防止某些浏览器无法自动播放音乐.取消监听事件
                document.removeEventListener("touchstart", self.touchendVoice);
                if (self.isRoomOwner) {
                  var host = window.location.host;
                  var allhref = window.location.href;
                  allhref = allhref.substr(0, allhref.length - 2);
                  var configid = self.$route.params.configid;
                  host = host + "/Music/" + configid;
                  var hint =
                    "复制下面的邀请链接发送给好友打开即可开始一起欣赏美妙的音乐";
                  if (
                    allhref.indexOf("localhost") != -1 ||
                    allhref.indexOf("0.0.0.0") != -1 ||
                    allhref.indexOf("127.0.0.1") != -1
                  ) {
                    hint = "复制下面的链接给好友输入到主页即可连接";
                  }
                  swal({
                    type: "success",
                    title: hint,
                    text: allhref,
                    confirmButtonText: "复制完成"
                  });
                }

                self.startInterval();
              } else {
                swal({
                  type: "error",
                  html: "歌单不存在"
                });
                self.$route.push("/");
              }
            }
          );
        } else {
          swal({
            type: "error",
            html: "获取数据失败"
          });
          self.isGetDataComplete = false;
        }
      });
    },
    //结束延迟执行
    enddealy: function() {
      this.isEnd = true;
    },
    //启动数据相关的定时器
    startInterval: function() {
      var self = this;
      this.firstopen = false;
      this.intervalObj = setInterval(function() {
        if (self.isEnd == false) {
          self.getJson();
        } else {
          //如果是true的话延迟6秒等待跳转完成之后在进行操作.
          setTimeout(function() {
            self.isEnd = false;
            //立即请求现在的数据
            self.sendJson();
          }, 4000);
        }
      }, 4 * 1000);
      setTimeout(function() {
        //如果没有找到aplyer对象.则不执行监听器
        if (
          self.$refs.musicplayer != null &&
          self.$refs.musicplayer != undefined
        ) {
          self.player = self.$refs.musicplayer.$children;
          self.setEventListener();
        } else {
          console.log("没有找到musicplayer");
        }
      }, 500);
    },
    setplay: function() {
      this.$refs.musicplayer.play();
      this.sendJson();
    },
    setpause: function() {
      this.$refs.musicplayer.pause();
      this.sendJson();
    },
    setPlayTime(time) {
      var audio = document.getElementsByTagName("audio")[0];
      audio.currentTime = time;
    },
    //设置切换的音乐index
    setPlayIndex(index) {
      this.$refs.musicplayer.onSelectSong(this.musiclist[index]);
    },
    //得到当前播放的索引
    getPlayIndex: function() {
      return this.player[2]._props.playIndex;
    },
    //得到当前播放时间
    getPlayTime: function() {
      var arr = new Array();
      //总时长
      arr[0] = this.player[1]._props.stat.duration;
      //加载时长
      arr[1] = this.player[1]._props.stat.loadedTime;
      //播放时长
      arr[2] = this.player[1]._props.stat.playedTime;
      return arr;
    },
    //获取播放模式 循环播放 暂停播放 单曲循环
    // getPlayMode: function() {
    //   var internalRepeat = this.$refs.musicplayer.internalRepeat;
    //   return internalRepeat;
    // },
    getPlayStatus: function() {
      return String(this.$refs.musicplayer.isPlaying);
    },
    //获取是否是随机播放(true)
    getPlayRandom: function() {
      var internalShuffle = this.$refs.musicplayer.internalShuffle;
      console.log(internalShuffle);
      return internalShuffle;
    },
    //点击同意按钮时,触发无声的音乐和请求数据
    agree: function() {
      //初始化获取传递过来的参数
      var configid = this.$route.params.configid;
      //判断获取过来的是不是管理员
      var admin = this.$route.params.admin;
      if (admin == "1") {
        //说明是房主开房
        this.isRoomOwner = true;
      }
      if (configid == undefined || configid == "" || configid == null) {
        this.$route.push("/");
      }
      //获取到配置文件地址
      this.configurl = "https://api.myjson.com/bins/" + configid;
      //初始化数据
      this.initData();
    },
    //获取返回的json数据
    getJson: function() {
      var self = this;
      if (self.musiclist == null || self.music == null) {
        return;
      }
      if (self.isRoomOwner) {
        return;
      }
      axios
        .get(self.configurl, {})
        .then(function(response) {
          var data = response.data;
          data = eval("(" + data.params.data + ")").sync;
          if (data.updatetime != undefined) {
            //说明数据没有初始化
            //说明当前播放的歌曲没有发生变化
            var othertime = parseFloat(data.time);
            //获取到总时长
            var totaltime = self.getPlayTime()[0];
            var nowtime = self.getPlayTime()[2];
            //获取上一个用户更新前触发的时间
            var updatetime = parseFloat(data.updatetime);
            //大概计算出网络传输花了多长时间(秒)
            var ctime = new Date().getTime();
            var usingsecond = (new Date().getTime() - updatetime) / 1000;
            //获取修改用户当前应该播放的时间
            var realtime = othertime + usingsecond;
            if (
              (data.index == 0 && self.getPlayIndex() == -1) ||
              (data.index == -1 && self.getPlayIndex() == 0) ||
              data.index == self.getPlayIndex()
            ) {
              //如果暂停了.那么我也暂停.如果播放了.我也从可以播放的地方开始播放
              if (self.getPlayStatus() != data.status) {
                  // self.setPlayTime(realtime);
                  //只有非房主才会收到影响
                  console.log(data.status);
                  if (data.status == "true") {
                    console.log("播放")
                    self.setplay();
                    if (Math.abs(realtime - nowtime) >= 3){
                      self.setPlayTime(realtime);
                    }
                  } else {
                    console.log("暂停")
                    self.setpause();
                  }
              } else {
                if (Math.abs(realtime - nowtime) >= 3 && self.getPlayStatus()!="false") {
                  self.setPlayTime(realtime);
                }
              }
            } else {
              if (self.preSwitchTime == null || self.preSwitchTime == "") {
                self.preSwitchTime = new Date().getTime();
              } else {
                //算出经过的秒数
                var second = (new Date().getTime() - self.preSwitchTime) / 1000;
                if (second > 3) {
                  //如果已经大于六秒中还是和当前的播放歌曲不同的话
                  //那就进行切换歌曲的操作
                  self.preSwitchTime = new Date().getTime();
                  //playIndex这里存在问题会导致暂停
                  self.setPlayIndex(data.index);
                  // self.$refs.musicplayer.play();
                }
              }
            }
          }
        })
        .catch(function(error) {
          swal({
            title: "请求异常",
            text: error,
            type: "error"
          });
        });
      self.sendJson();
    },
    //修改json数据
    sendJson: function() {
      if (
        this.songid == "" ||
        this.songid == undefined ||
        this.songid == null
      ) {
        console.log("发现歌单id不存在!");
        return;
      }

      var self = this;
      //判断现在是不是房主.如果是房主则就发送改变.
      if (!this.isRoomOwner) {
        return;
      }
      //获取上次事件触发的时间
      var currenttime = new Date().getTime();
      if (self.presendtime == null) {
        self.presendtime = currenttime;
      } else {
        var spacetime = (currenttime - self.presendtime) / 1000;
        if (spacetime <= 2) {
          self.presendtime = currenttime;
          return;
        }
      }
      self.playtime = self.getPlayTime()[2];
      self.playindex = self.getPlayIndex();
      self.playstatus = self.getPlayStatus();
      //说明是第一次触发事件
      //获取到当前的时间戳
      axios
        .put(self.configurl, {
          params: {
            data:
              '{"sync":{"listid":"' +
              self.songid +
              '","time":"' +
              self.getPlayTime()[2] +
              '","index":"' +
              self.getPlayIndex() +
              '","mode":"' +
              "none" +
              '","status":"' +
              self.getPlayStatus() +
              '","updatetime":"' +
              currenttime +
              '"}}'
          }
        })
        .then(function(response) {
          // console.log(response.data);
        })
        .catch(function(error) {});
    },
    //发送当前的状态
    sendCurrentJson: function() {
      this.getJson();
    },
    //接触屏幕触发监听事件
    touchendVoice: function() {
      var musicEle0 = document.getElementById("emitaudio");
      musicEle0.play();
    }
  },
  data() {
    return {
      //当前歌曲id
      songid: "",
      //当前音乐列表
      musiclist: null,
      //当前音乐
      music: null,
      //获取数据是否完成
      isGetDataComplete: false,
      //控制器样式
      controllerstyle: "hidden-controller",
      //是否是刚开屏
      firstopen: true,
      //配置文件地址
      configurl: "",
      //播放进度
      playtime: "",
      //播放索引
      playindex: "",
      //播放模式
      playmode: "",
      //播放状态
      playstatus: "",
      //随机码
      random: "123",
      pre_event_time: 0,
      intervalObj: null,
      isEnd: false,
      presendtime: null,
      //如果是房主的话就不会隐藏进度条.
      //目前的想法是只有房主才能进行进度的调整.而客户端仅仅只是接受信息而已
      isRoomOwner: false
    };
  },
  mounted() {
    //防止某些浏览器无法自动播放音乐.取消监听事件
    document.addEventListener("touchend", this.touchendVoice);
    // 创建房间
    // this.createJson()
  },
  beforeDestroy() {
    //取消所有的监听事件
    this.cancleEventListener();
    //取消定时器
    clearInterval(this.intervalObj);
  }
};
</script>

<style lang="stylus">
#HelloDiv {
  position: fixed;
  color: white;
  left: 0;
  top: 0;
  bottom: 0;
  right: 0;
  background-color: black;

  #agereeButton {
    font-size: 2rem;
    margin-top: 20rem;
    text-align: center;
  }
}

// 隐藏控制栏
.hidden_controller {
  .aplayer-controller {
    visibility: hidden;
  }
}
</style>
