<template>
    <div class="wrapper">
        <myheader
            :speed="speed"
            :isPlaying="isPlaying"
            ></myheader>
        <painter :src="src"></painter>
        <myfooter 
            :frame_total="frame_total"
            :play_frame_idx = "play_frame_idx"
            :isPlaying="isPlaying"
            ></myfooter>
    </div>
</template>

<script>
import myheader from './components/Header.vue'
import painter from './components/painter.vue'
import myfooter from './components/Footer.vue'

import { getSrc } from './services/getSrc.js'
var loader = require('./assets/js/loader.js')
export default {
    name: 'app',
    data () {
        return {
            src:'',
            isPlaying:false,
            frames:[],
            speed:50,
            interval:0,
            delay:2,//ms
            frame_total:10,
            play_frame_idx:0,
            dsa_path:''
        }
    },
    mounted(){

        window.VM = this; // 暴露自己的API 给全局

        let self = this;
        self.dsa_path=self.$route.params.id;
        //事件监听
        self.$eventHub.$on('setPlayFrameIdx',self.setPlayFrameIdx);
        self.$eventHub.$on('setSpeed',self.setSpeed);
        self.$eventHub.$on('toggle_play',self.toggle_play);
        console.log("开始加载DAS数据:--");
        console.log("要加载的DAS数据地址是:/"+self.dsa_path);
        let base_url = 'dsa/'+self.dsa_path+"/";
        let app = this.app;
        Promise.all([
            getSrc(base_url+"algorithm.c"),
            app.loadScript(base_url+'render.js'),
            app.loadScript(base_url+'line.js'),
            app.loadStyle(base_url+'style.css')
        ]).then(function(data){
            console.log('DSA数据加载完毕!')
            self.src = data[0];
        }).then(function(){ // 初始化render.js
            self.__init__() //初始化
            //设定播放
            setInterval(self.__play,25);
        })
    },
    methods:{
      __init__:function(re_init){
            console.log("运行_reader_init");
            if(!re_init)
              render_init();
            this.frames = lineExports.init();
            console.log('生成frames数据完成!');
            //设定frame宽度
            this.frame_total = self.frames.length;
            //播放第一帧
            this.play_frame_idx = 1;
      },
      re_init(){
        this.__init__(true)
        render(this.frames[0].status,speedScale(this.speed));
      },
        hl:function(s,t){ //高亮行
            this.$eventHub.$emit('hl-lines',s,t);
        },
        setPlayFrameIdx:function(idx){
            if(idx < 1)
                idx = 1;
            else if(idx > this.frame_total)
                idx = this.frame_total;
            this.play_frame_idx = idx;
        },
        setSpeed:function(idx){
            if(idx <1)
                idx = 1;
            else if( idx > 100)
                idx = 100;
            this.speed= idx;
            console.log('当前速度是:'+idx)
            console.log('当前真实速度是:'+speedScale(idx) +"ms")
        },
        toggle_play:function(){
            this.isPlaying = this.isPlaying?false:true;
        },
        __play:function(){
            let self = this;
            let speed_time = speedScale(self.speed);
            self.interval += 25; // 25ms
            if( self.interval >+(self.delay + speed_time)){
                if(self.isPlaying){
                    if(self.play_frame_idx+1 <= self.frame_total){
                        self.play_frame_idx++;
                    }
                    else {
                        //self.play_frame_idx = 1;
                        self.isPlaying = false;
                    }
                }
                self.interval = 0;
            }
        },
        __sendMessage:function(message){
            this.$eventHub.$emit('sendMessage',message);
        },
        __clearMessage:function(){
            this.$eventHub.$emit('clearMessage');
        }
    },
    watch:{
        play_frame_idx:function(newValue,oldValue){
          let  vm = this;
          console.log('从第',oldValue,'帧到-->第',newValue,'帧');
          render(vm.frames[newValue-1].status,speedScale(vm.speed));
          vm.hl(vm.frames[newValue-1].hls,vm.frames[newValue-1].hlt);
          if(vm.frames[newValue-1].log !== ''){
            vm.__sendMessage(vm.frames[newValue-1].log,500);
          }
        }
    },
    components:{
        myheader,
        painter,
        myfooter
    }
}
</script>
