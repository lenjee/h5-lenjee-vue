<template>
  <div class="detail">
    <div class="frm-list">
      <yd-cell-group title="">
        <yd-cell-item>
            <yd-textarea slot="right" v-model='content' placeholder="举报内容:" maxlength="200"></yd-textarea>
        </yd-cell-item>
      </yd-cell-group>
    </div>

    <div class="frm-list">
      <yd-cell-group>
        <yd-cell-item>
            <span slot="left">从地图中选择：</span>
            <span slot="right">
              <div @click='showMap()' class="frm-list-right">
                <img src="@/assets/img/map.png" alt="">
                <div class="frm-addr-txt">{{wx_addr}}</div>
              </div>
            </span>
        </yd-cell-item>
      </yd-cell-group>
    </div>

    <div class="frm-list">
      <yd-cell-group>
        <yd-cell-item>
            <span slot="left">详细地址：</span>
            <input slot="right" type="text" v-model='user_addr' placeholder="">
        </yd-cell-item>
      </yd-cell-group>
    </div>

    <div class="frm-list bottom-line">
      <div class="frm-list-head">
        <div class="frm-list-head-l">
          现场图片：
        </div>
        <div class="frm-list-head-r">
          {{img.length}}/4
        </div>
      </div>
      <div class="frm-list-img">
        <div class="frm-list-img-l _bg">
          <input type="file" @change="uploadFile('img')" class="input_file" ref="imgFile" name="" value="">
        </div>

        <div class="frm-list-img-l" v-for='(item,i) in img'>
          <img :src="item" alt="">
          <div class="close" @click='close(i,0)'></div>
        </div>

      </div>
    </div>

    <div  class="frm-list bottom-line">
      <div class="frm-list-head">
        <div class="frm-list-head-l">
          现场视频：
        </div>
        <div class="frm-list-head-r">
          {{video.length}}/1
        </div>
      </div>
      <div class="frm-list-img">
        <div class="frm-list-img-v">
          <input type="file" @change="uploadFile('video')" class="input_file" accept="video/*" capture="camera" ref="videoFile" name="" value="">
        </div>
        <div class="frm-list-img-l" v-for='(item,i) in video'>
          <video  class="video" :src="item">您的浏览器不支持 video 标签</video>
          <div class="close" @click='close(i,1)'></div>
        </div>
      </div>
    </div>

    <div class="frm-list">
      <yd-cell-group>
        <yd-cell-item>
            <span slot="left">您的性别：</span>
            <div slot="right" class="frm-list-rado">
              <yd-radio-group v-model="sex">
                <yd-radio val="1">男士</yd-radio>
                <yd-radio val="2">女士</yd-radio>
              </yd-radio-group>
            </div>
        </yd-cell-item>
      </yd-cell-group>
    </div>

    <div class="frm-list">
      <yd-cell-group class="phoneNum">
          <yd-cell-item>
              <span slot="left">手机号：</span>
              <yd-input slot="right" v-model="phone" required :show-success-icon="false" :show-error-icon="false" regex="mobile" placeholder="请输入手机号码"></yd-input>
          </yd-cell-item>
      </yd-cell-group>
    </div>

    <div @click="submit()" class="frm-btn">
      提交线索
    </div>

    <div  v-show="isShowMap" id="outer-box">
      <div @click='hideMap()' class="goback">
        <img src="@/assets/img/goback.png" alt=""> <span>确定</span>
      </div>
      <div id="container"></div>
      <div id="panel" class="scrollbar1">
          <div id="searchBar">
              <input id="searchInput" placeholder="输入关键字搜素POI" />
          </div>
          <div id="searchResults"></div>
      </div>
    </div>

  </div>
</template>

<script type="text/babel">
// 请求
import {  AjaxGet , AjaxPostForm , AjaxPostJson} from '@/service/ajax.js';
// 注册 js-sdk
import { RegistJsSdk } from '@/service/wechat_jssdk.js';
// 高德地图组件
import AMap from 'AMap';
import AMapUI from 'AMapUI';

// 缓存
import Cache from '@/util/cache';
let map_skyin,marker_skyin,_this;

export default {
  name: 'Skyin-Search',
  data () {
    return {
      isShowMap:false,
      content:'',  // 举报内容
      phone:'',  // 手机号
      wx_addr:'点击选择', // 地理位置
      user_addr:'', // 详细地址
      sex: '1', // 性别
      img:[] ,// 图片地址 最多四张
      video:[], // 视频地址
    }
  },
  activated(){
    this.initPage();
  },
  methods: {
    initPage:function(){
      _this=this;

      // 注册 js-sdk
      let lcUrl=window.location.href;
      RegistJsSdk(this,lcUrl,()=>{
        console.log("##### 微信初始化jssdk成功 calllback #####");
        _this.initAmap();
      });

      // this.TestFun();
    },
    close(i,type){//删除图片
      if(type ==0){
        this.img.splice(i,1);
        console.log(this.img);
     }else {
        this.video.splice(i,1);
        console.log(this.img);
     }
    },
    hideMap:function(){ // 隐藏地图
      this.isShowMap=false;
    },
    showMap:function(){
      this.isShowMap=true;
    },
    initWehatMap:function(){
      console.log("*********************************");
      console.log("微信 初始化 地理位置");
      console.log("*********************************");

      // _this.getLoaction('116.397428','39.90923');
      window.wx.ready(function () {

          // 1 判断当前版本是否支持指定 JS 接口，支持批量判断
          window.wx.checkJsApi({
            jsApiList: [
              'getNetworkType',
              'previewImage'
            ],
            success: function (res) {
              console.log('***** 判断当前版本是否支持指定 JS 接口，支持批量判断 *****');
              let wxld=JSON.stringify(res);
              console.log(wxld);

              // 7.2 获取当前地理位置
              window.wx.getLocation({
                type: 'wgs84', // 默认为wgs84的gps坐标，如果要返回直接给openLocation用的火星坐标，可传入'gcj02'
                success: function (res) {
                  console.log('***** 微信 获取当前地理位置 *****');
                  console.log(res);
                  let longitude=res.longitude;
                  let latitude=res.latitude;
                  // 初始化高德地图
                  _this.setAddr(longitude,latitude);
                },
                cancel: function (res) {
                  _this.getLoaction('116.397428','39.90923');
                  _this.$dialog.alert({mes: '用户拒绝授权获取地理位置'});
                }
              });

            }
          });

      });
      window.wx.error(function (res) {
        console.log('***** JS-SDK 注册失败 *****');
        console.log(res);
        // _this.getLoaction('116.397428','39.90923');
        _this.wx_addr = '获取地理信息失败';
      });

    },

    // openCamera:function(){
    //   console.log("*********************************");
    //   console.log("获取图片");
    //   console.log("*********************************");
    //
    //   if(_this.img.length>=4){
    //     console.log('最多上传四张');
    //     return;
    //   }
    //
    //   let ck_ticket=Cache.cookie.get("ticket");
    //   let ck_openid=Cache.cookie.get("openid");
    //
    //   // loading
    //   _this.$dialog.loading.open('正在上传...');
    //   // 关闭 loading
    //   _this.$dialog.loading.close();
    //
    //   return;
    //   window.wx.ready(function () {
    //       // 1 判断当前版本是否支持指定 JS 接口，支持批量判断
    //       window.wx.checkJsApi({
    //         jsApiList: [
    //           'getNetworkType',
    //           'previewImage'
    //         ],
    //         success: function (res) {
    //
    //           console.log('***** 判断当前版本是否支持指定 JS 接口，支持批量判断 *****');
    //           console.log(JSON.stringify(res));
    //
    //           // 5.1 拍照、本地选图
    //           let images = {
    //             localId: [],
    //             serverId: []
    //           };
    //           window.wx.chooseImage({
    //             success: function (res) {
    //               console.log('***** 拍照、本地选图 ***** : ',res);
    //
    //               images.localId = res.localIds;
    //               if (images.localId.length > 4) {
    //                    _this.$dialog.alert({mes: '最多只能选择四张图片'});
    //                    return;
    //               }
    //
    //
    //               // 5.3 上传图片
    //               let i = 0, length = images.localId.length;
    //               function upload() {
    //                 wx.uploadImage({
    //                   localId: images.localId[i], // 需要上传的图片的本地ID，由chooseImage接口获得
    //                   isShowProgressTips: 1, // 默认为1，显示进度提示
    //                   success: function (res) {
    //                     console.log('uploadImage -> 上传返回值: ',res);
    //                     i++;
    //                     images.serverId.push(res.serverId);  // res.serverId 是 返回图片的服务器端ID
    //
    //                     let localData = res.localData; // localData是图片的base64数据，可以用img标签显示
    //
    //                     let url=window.__APPINFO__.host+"/home/tool/upload";
    //
    //                     let formdata = new FormData();
    //                     formdata.append('file',localData); // 图片或者视频的字段
    //                     formdata.append('path',"img"); // 存储的路由; 图片上传时，path = img ; 视频上传时，path = vedio;
    //                     formdata.append('ticket',ck_ticket);
    //                     formdata.append('openid',ck_openid);
    //
    //
    //                     // 上传到服务器
    //                     AjaxPostForm(_this,url,formdata,(res)=>{
    //                       console.log('上传到服务器 -> 返回值 : ', res );
    //                       // 关闭 loading
    //                       _this.$dialog.loading.close();
    //                       if(res.data.code==0){
    //                         console.log('已上传：' + i + '/' + length);
    //                         _this.img.push(res.data.result.url);
    //                       }else{
    //                         console.log('第：' + i + '/' + length+'上传失败');
    //                         _this.$dialog.toast({
    //                             mes:'上传失败',
    //                             timeout: 1500,
    //                             icon: 'error'
    //                         });
    //                       }
    //
    //                     });
    //
    //                     if (i < length) {
    //                       upload();
    //                     }
    //
    //                   },
    //                   fail: function (res) {
    //                     console.log('微信上传报错: ',res);
    //                   }
    //                 });
    //               }
    //               upload();
    //             }
    //           });
    //
    //         }
    //       });
    //
    //   });
    //
    //   window.wx.error(function (res) {
    //     console.log('***** JS-SDK 注册失败 *****');
    //     console.log(res);
    //   });
    // },
    // openMedia:function(){
    //   console.log("*********************************");
    //   console.log("获取摄像");
    //   console.log("*********************************");
    //
    //   if(_this.img.length>=1){
    //     console.log('最多上传1张');
    //     return;
    //   }
    //
    //   let ck_ticket=Cache.cookie.get("ticket");
    //   let ck_openid=Cache.cookie.get("openid");
    //
    //   window.wx.ready(function () {
    //
    //       // 1 判断当前版本是否支持指定 JS 接口，支持批量判断
    //       window.wx.checkJsApi({
    //         jsApiList: [
    //           'getNetworkType',
    //           'previewImage'
    //         ],
    //         success: function (res) {
    //
    //           console.log('***** 判断当前版本是否支持指定 JS 接口，支持批量判断 *****');
    //           console.log(JSON.stringify(res));
    //
    //           // 5.1 拍照、本地选图
    //           let images = {
    //             localId: [],
    //             serverId: []
    //           };
    //           window.wx.chooseImage({
    //             success: function (res) {
    //               console.log('***** 拍照、本地选图 *****');
    //               images.localId = res.localIds;
    //               if (images.localId.length > 1) {
    //                    _this.$dialog.alert({mes: '最多只能选择1个视频'});
    //                    return;
    //               }
    //               // 5.3 上传图片
    //               let i = 0, length = images.localId.length;
    //               function upload() {
    //                 wx.uploadImage({
    //                   localId: images.localId[i], // 需要上传的图片的本地ID，由chooseImage接口获得
    //                   isShowProgressTips: 1, // 默认为1，显示进度提示
    //                   success: function (res) {
    //                     console.log('uploadImage -> 上传返回值: ',res);
    //                     i++;
    //                     images.serverId.push(res.serverId);  // res.serverId 是 返回图片的服务器端ID
    //
    //                     let localData = res.localData; // localData是图片的base64数据，可以用img标签显示
    //
    //                     let url=window.__APPINFO__.host+"/home/tool/upload";
    //
    //                     let formdata = new FormData();
    //                     formdata.append('file',localData); // 图片或者视频的字段
    //                     formdata.append('path',"vedio"); // 存储的路由; 图片上传时，path = img ; 视频上传时，path = vedio;
    //                     formdata.append('ticket',ck_ticket);
    //                     formdata.append('openid',ck_openid);
    //
    //                     // 上传到服务器
    //                     AjaxPostForm(_this,url,formdata,(res)=>{
    //                       console.log('上传到服务器 -> 返回值 : ', res );
    //                       if(res.data.code == 0 ){
    //                         console.log('已上传：' + i + '/' + length);
    //                         _this.img.push(res.data.result.url);
    //                       }else{
    //                         console.log('第：' + i + '/' + length+'上传失败');
    //                         _this.$dialog.toast({
    //                             mes:'上传失败',
    //                             timeout: 1500,
    //                             icon: 'error'
    //                         });
    //                       }
    //
    //                     })
    //
    //                     if (i < length) {
    //                       upload();
    //                     }
    //
    //                   },
    //                   fail: function (res) {
    //                     _this.$dialog.alert({mes: '微信打开图片出错，请重试'});
    //                     console.log('微信打开图片出错: ',JSON.stringify(res));
    //                   }
    //                 });
    //               }
    //               upload();
    //
    //             }
    //           });
    //
    //         }
    //       });
    //
    //   });
    //
    //   window.wx.error(function (res) {
    //     console.log('***** JS-SDK 注册失败 *****');
    //     console.log(res);
    //   });
    // },

    dataValidation:function(){
      let rtn={
        code:true,
        msg:'成功'
      }
      if(this.content==""){
        rtn.code=false;
        rtn.msg = '举报内容不能为空';
        return rtn ;
      }
      if(this.wx_addr=="获取地理信息失败"||this.wx_addr=="点击选择" ){
        rtn.code=false;
        rtn.msg = '请选择地理位置';
        return rtn ;
      }
      if(this.img.length==0){
        rtn.code=false;
        rtn.msg = '请上传图片';
        return rtn ;
      }
      if(this.phone==""){
        rtn.code=false;
        rtn.msg = '手机号不能为空';
        return rtn ;
      }
      return rtn;
    },
    submit:function(){
      console.log("--------- 提交数据 ----------");

      let isDataOk=_this.dataValidation();

      if(!isDataOk.code){
        _this.$dialog.toast({
            mes: isDataOk.msg,
            timeout: 1500,
            icon: 'error'
        });
        return;
      }

      let ck_ticket=Cache.cookie.get("ticket");
      let ck_openid=Cache.cookie.get("openid");
      let cname=Cache.localStorage.get('type');
      let _data={
        img_list:this.img.join("|"), // 图片地址 数组最多四张
        desc:this.content,// 举报内容
      	"video_list":this.video.join("|"), // 视频地址
      	"wx_addr":this.wx_addr,
      	"user_addr":this.user_addr,
      	"sex":this.sex, // 性别
      	"phone":this.phone,  // 手机号
      	"cname":cname, // 举报类型
        "openid":ck_openid,
      	"ticket":ck_ticket
      }
      console.log("提交的数据: ",_data);

      let url=window.__APPINFO__.host+"/home/cms/add";
      // 上传到服务器
      AjaxPostJson(_this,url,_data,(res)=>{
        console.log('表单提交  -> 返回值 : ', res );

        if(res.data.code==0){
          _this.$router.push("/sucess");
        }else{
          _this.$dialog.alert({mes: 'res.data.msg'});
        }

      });

    },
    initAmap:function(){
      console.log("#####################");

      console.log("初始化 高德地图  ");
      console.log("******** 由于Chrome、IOS10等已不再支持非安全域的浏览器定位请求，为保证定位成功率和精度，请尽快升级您的站点到HTTPS。******");

      let geolocation;
      let ck_Lng,ck_Lat;


      // 初始定位
      map_skyin = new AMap.Map('container', {
        resizeEnable: true,
        zoom:16,//级别
      });

      // 获取当前地址
      // map_skyin.plugin(['AMap.Geolocation'], ()=> {
      //   geolocation = new AMap.Geolocation({
      //     enableHighAccuracy: true, //  是否使用高精度定位，默认:true
      //     timeout: 5000, //  超过5秒后停止定位，默认：无穷大
      //     maximumAge: 0, // 定位结果缓存0毫秒，默认：0
      //     convert: true, // 自动偏移坐标，偏移后的坐标为高德坐标，默认：true
      //     showButton: true, //  显示定位按钮，默认：true
      //     buttonPosition: 'LB',  // 定位按钮停靠位置，默认：'LB'，左下角
      //     buttonOffset: new AMap.Pixel(10, 20), //  定位按钮与设置的停靠位置的偏移量，默认：Pixel(10, 20)
      //     showMarker: true, //  定位成功后在定位到的位置显示点标记，默认：true
      //     showCircle: true, //  定位成功后用圆圈表示定位精度范围，默认：true
      //     panToLocation: true,  //  定位成功后将定位到的位置作为地图中心点，默认：true
      //     zoomToAccuracy: true  //  定位成功后调整地图视野范围使定位位置及精度范围视野内可见，默认：false
      //   })
      //   map_skyin.addControl(geolocation);
      //   geolocation.getCurrentPosition()
      //   AMap.event.addListener(geolocation, 'complete', (result) => {
      //     // //  返回定位信息
      //     // _this.map.setCenter(result.position);
      //     // console.log('经纬度：',result.position);
      //     // console.log('精度范围：',result.accuracy);
      //     // console.log('定位结果的来源：'+result.location_type);
      //     // console.log('状态信息：',result.info);
      //     // console.log('地址：',result.formattedAddress);
      //     // console.log('地址信息：',JSON.stringify(result.addressComponent, null, 4));
      //     _this.wx_addr=result.formattedAddress;
      //   });
      //   AMap.event.addListener(geolocation, 'error', (result) => {
      //     //  返回定位出错信息
      //     _this.wx_addr = '获取地理信息失败';
      //     console.log("返回定位出错信息 -> : ",result)
      //   });
      // });

      // 标记
      marker_skyin = new AMap.Marker()
      map_skyin.add(marker_skyin);//添加到地图

      // 为地图添加点击事件并标记
      map_skyin.on('click', (e)=> {
        ck_Lng=e.lnglat.getLng();
        ck_Lat=e.lnglat.getLat();

        _this.setAddr(ck_Lng,ck_Lat)

      });

      // 插件
      map_skyin.plugin(['AMap.ToolBar', 'AMap.MapType'], ()=>{
        map_skyin.addControl(new AMap.ToolBar())
        map_skyin.addControl(new AMap.MapType({showTraffic: false, showRoad: false}))
      });

      // 初始化 地址
      _this.initWehatMap();

    },
    setAddr:function(ck_Lng,ck_Lat){ // 设置地址
      console.log("设置地址：ck_Lng：",ck_Lng," ck_Lat:",ck_Lat);

      let map_key="";
      // 移除已创建的 marker
      map_skyin.remove(marker_skyin);
      // 创建新的标记
      marker_skyin = new AMap.Marker({
        position:[ck_Lng,ck_Lat]//位置
      })
      map_skyin.add(marker_skyin);

      AMap.service('AMap.Geocoder',()=>{//回调函数
           //实例化Geocoder
           let geocoder = new AMap.Geocoder({
               radius: 1000,
               extensions: "all"
           });
           let lnglatXY=[ck_Lng, ck_Lat];//地图上所标点的坐标
           geocoder.getAddress(lnglatXY, function(status, result) {
             console.log("地址结果： status: ",status ," result: ",result);
             if (status === 'complete' && result.info === 'OK') {
                 map_key = result.regeocode.formattedAddress;
                 _this.wx_addr = result.regeocode.formattedAddress;

                 // 数据条目
                 AMapUI.loadUI(['misc/PoiPicker'], function(PoiPicker) {
                   let poiPicker = new PoiPicker({
                      input: 'searchInput',
                      placeSearchOptions: {
                          map: map_skyin,
                          pageSize:4
                      },
                      searchResultsContainer: 'searchResults'
                   });

                   poiPicker.on('poiPicked', function(poiResult) {
                      poiPicker.hideSearchResults();
                      let source = poiResult.source,
                          poi = poiResult.item;
                      if (source !== 'search') {
                        //suggest来源的，同样调用搜索
                        poiPicker.searchByKeyword(poi.name);
                      } else {
                        let poi_wx_addr = result.regeocode.formattedAddress+"-"+poi.name;
                        console.log("poi: ",poi_wx_addr);
                        _this.wx_addr =poi_wx_addr;
                      }
                   });

                   poiPicker.onCityReady(function() {
                      poiPicker.searchByKeyword(map_key);
                   });

                 });


             }else{
                 _this.wx_addr = '获取地理信息失败';
             }
           });
       });

    },
    uploadFile:function(e){
      console.log('### uploadFile --> : ',e)
      let _this=this , useFile;
      let ck_ticket=Cache.cookie.get("ticket");
      let ck_openid=Cache.cookie.get("openid");

      if(e=='img'){
        if(_this.img.length>=4){
          _this.$dialog.alert({mes: '最多上传四张'});
          return;
        }
        useFile=_this.$refs.imgFile.files[0];

        console.log('input.file.files : ',useFile)
        console.log('file type : ',useFile.type)

        let fileType=useFile.type.split('/')[1];
        if(fileType=='jpeg'||fileType=='jpg'||fileType=='png'){
          console.log("格式正确")
        }else{
          _this.$dialog.alert({mes: '上传图片的格式有误，请上传正确格式的图片'});
          return;
        }
      }

      if(e=='video'){
        if(_this.video.length>=1){
          _this.$dialog.alert({mes: '最多上传一个视频'});
          return;
        }
        useFile=_this.$refs.videoFile.files[0];
        console.log('input.file.files : ',useFile)
        console.log('file type : ',useFile.type)

        let fileType=useFile.type.split('/')[1];
        if(fileType=='mov'||fileType=='mp4'||fileType=='quicktime'){
          console.log("视频格式正确")
        }else{
          _this.$dialog.alert({mes: '上传视频格式有误，请上传正确格式的视频'});
          return;
        }
      }


      // loading
      _this.$dialog.loading.open('正在上传...');

      let url=window.__APPINFO__.host+"/home/tool/upload";

      let formdata = new FormData();
      formdata.append('file',useFile); // 图片或者视频的字段
      formdata.append('path',e); // 存储的路由; 图片上传时，path = img ; 视频上传时，path = vedio;
      formdata.append('ticket',ck_ticket);
      formdata.append('openid',ck_openid);

      // 上传到服务器
      AjaxPostForm(_this,url,formdata,(res)=>{
        // 关闭 loading
        _this.$dialog.loading.close();
        console.log('上传到服务器 -> 返回值 : ', res );
        if(res.data.code == 0 ){
          let addImgUrl=window.__APPINFO__.host+res.data.result.url;
          if(e=='img'){
            _this.img.push(addImgUrl);
          }
          if(e=='video'){
            _this.video.push(addImgUrl);
          }
          console.log("***** img: ", _this.img );
          console.log("***** video: ",_this.video );
        }else{
          _this.$dialog.toast({
              mes:'上传失败',
              timeout: 1500,
              icon: 'error'
          });
        }
      });

    },
  }
}
</script>

<style  lang="less" rel="stylesheet/less">
  @import "./index.less";
</style>
