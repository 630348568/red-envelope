<template>
  <div class="more_play_box">
   <div class="play_box_center">
     <div class="head_img">
       <img :src="headImg" alt="">
     </div>
     <div class="choose_item" @click="chooseMaterialData()">
       <div class="item_title">
         选择图片
       </div>
       <div class="item_type">
         <img :src="picData" alt="">
       </div>
       <div class="next_step_choose">
         >
       </div>
     </div>
     <div class="choose_tips">
       小伙伴们拼对图片即可获得随机红包
     </div>
     <div class="edit_item">
       <div class="item_title">红包金额</div>
      <div class="item_input"><input placeholder-class="place-holder" type='number' @change="fieldMoney" placeholder="填写金额" /></div>
      <div class="next_step">元</div>
    </div>
    <div class="edit_item">
       <div class="item_title">红包数量</div>
      <div class="item_input"><input placeholder-class="place-holder" type='number' @change="fieldNumber" placeholder="填写数量" /></div>
      <div class="next_step">个</div>
    </div>
    <div class="edit_item" @click="pickerTimeData()">
      <div class="item_title">挑战时间</div>
      <div class="item_input"><input :value='timeSeconds' placeholder-class="place-holder" disabled placeholder="选择时间" /></div>
      <div class="next_step_choose">></div>
    </div>
    <div class="edit_item">
       <div class="item_title">挑战难度</div>
      <div class="item_input">
        <picker class="picker_box" @change="pickerLevelChange" :value="levelIndex" :range="levelArray">
          <div class="picker_level">
            {{levelArray[levelIndex]}}
          </div>
        </picker>
      </div>
      <div class="next_step_choose">></div>
    </div>
    <div class="edit_item">
      <div class="item_title">发布广场</div>
      <div class="item_publish"><switch checked @change="switchChange"/></div>
    </div>
    <div class="fee_info">
      需支付¥{{needFee}}元服务费，优先使用余额¥{{balance}}元。
    </div>
   </div>
   <div class="save_as_package" @click="makePicPackage()">
     生成拼图红包
   </div>
   <pickerTime :isShowTimeModal='isShowTimeModal' @setTimeCallBack="setTimeCallBack"></pickerTime>
  </div>
</template>

<script>
import pickerTime from '../../../components/modal/pick-time/index'
export default {
  components: {
    pickerTime
  },

  data () {
    return {
      levelArray: ['3 X 3', '4 X 4', '5 X 5'],
      levelIndex: 0,
      levelMap: {
        0: 1,
        1: 2,
        2: 3
      },
      timeSeconds: '',
      timeIndex: 0,
      isShowTimeModal: false,
      money: 0,
      needFee: 0,
      num: '',
      balance: 0,
      headImg: '../../../static/images/test_img.png',
      picData: '',
      isPublish: 1
    }
  },
  methods: {
    chooseMaterialData () {
      let parmas = null
      wx.navigateTo({
        url: `/pages/material/main?id=${parmas}&type=pic`
      })
    },
    switchChange (val) {
      // 是否发布到广场
      let publishVal = val.target.value
      if (publishVal) {
        this.isPublish = 1
        return
      }
      this.isPublish = 2
    },
    pickerLevelChange (val) {
      this.levelIndex = val.target.value
    },
    pickerTimeData () {
      this.isShowTimeModal = true
    },
    setTimeCallBack (val, status) {
      this.timeSeconds = val + 's'
      this.isShowTimeModal = false
    },
    fieldMoney (e) {
      this.money = e.target.value
      this.needFee = e.target.value * 0.02
    },
    fieldNumber (e) {
      this.num = e.target.value
    },
    pickerTimeChange (val) {
      this.timeIndex = val.target.value
    },
    makePicPackage () {
      let {money, num, contentId, timeSeconds, isPublish} = this
      if (!money || !num || !contentId || !timeSeconds || !isPublish) {
        wx.showToast({
          title: '请输入红包参数',
          icon: 'none'
        })
        return
      }
      // 生成拼图红包
      let postParams = {
        memberId: this.memberId,
        type: 1, // 1-拼图 2-拼字 3-语音
        contentId, // '素材id'
        money, // 红包
        num, // 红包个数
        timeRange: parseInt(timeSeconds), // 挑战时间
        bonusMoney: money * 0.02, // 手续费（按2%收取）
        publish: isPublish, // 1-发布 2-不发布
        level: this.levelMap[this.levelIndex] // 难度 1， 2， 3
      }
      // 调用应用实例的方法获取全局数据
      this.request.post('/api/sendOutRecord/commit', postParams).then(res => {
        let payType = res.data.payType
        let id = res.data.id
        let param = {
          id,
          memberId: this.memberId
        }
        if (payType === 1) {
          // 余额支付
          this.request.get('/api/sendOutRecord/payByBalance', param).then(res => {
            if (res.code === '200') {
              wx.navigateTo({
                url: `/pages/red-package/detail/main?type=1&id=${id}`
              })
            }
          }).catch(err => {
            console.log(err)
          })
          return false
        }
        let jsonobject = res.data.jsonobject
        let {timeStamp, nonceStr, packageStr, paySign} = jsonobject
        wx.requestPayment(
          {
            timeStamp,
            nonceStr,
            package: packageStr,
            signType: 'MD5',
            paySign,
            success: function (res) {
              if (res.errMsg === 'requestPayment:ok') {
                wx.navigateTo({
                  url: `/pages/red-package/detail/main?type=1&id=${id}`
                })
              } else {
                wx.showToast({
                  title: '支付失败，请重新支付',
                  icon: 'none',
                  duration: 2000
                })
              }
            },
            fail: function (res) {

            },
            complete: function (res) {

            }
          })
      }).catch(err => {
        console.log(err)
      })
    }
  },
  created () {
  },
  // onUpload () {
  //   console.log('我销毁了')
  // },
  // onHide () {
  //   console.log('我藏起来了')
  //   this.money = ''
  //   this.num = ''
  //   this.timeSeconds = ''
  //   this.picData = ''
  // },
  onShow () {
    this.levelArray = ['3 X 3', '4 X 4', '5 X 5']
    this.levelIndex = 0
    this.levelMap = {
      0: 1,
      1: 2,
      2: 3
    }
    this.timeSeconds = ''
    this.timeIndex = 0
    this.isShowTimeModal = false
    this.money = 0
    this.needFee = 0
    this.num = ''
    this.balance = 0
    this.headImg = '../../../static/images/test_img.png'
    this.picData = ''
    this.isPublish = 1
    // eslint-disable-next-line
    let pages = getCurrentPages()
    let curPage = pages[pages.length - 1] // 素材库选择的内容
    if (curPage.data.data) {
      this.picData = curPage.data.data
      this.contentId = curPage.data.contentId
    }
    let that = this
    this.memberId = wx.getStorageSync('memberId')
    let postParams = {
      memberId: this.memberId
    }
    // 获取余额
    this.request.get('/api/sys/config/memberInfo', postParams).then(res => {
      that.balance = res.data.money
      that.headImg = res.data.headImg
    }).catch(err => {
      console.log(err)
    })
  },
  onUnload () {
    this.levelArray = ['3 X 3', '4 X 4', '5 X 5']
    this.levelIndex = 0
    this.levelMap = {
      0: 1,
      1: 2,
      2: 3
    }
    this.timeSeconds = ''
    this.timeIndex = 0
    this.isShowTimeModal = false
    this.money = 0
    this.needFee = 0
    this.num = ''
    this.balance = 0
    this.headImg = '../../../static/images/test_img.png'
    this.picData = ''
    this.isPublish = 1
  }
}
</script>

<style>
.more_play_box{
  height: 100%;
  width: 100%;
  position: fixed;
  background: #f4f4f4;
}
.play_box_center{
  position: relative;
  margin: 198rpx 60rpx 0 60rpx;
  background:rgba(255,255,255,1);
  box-shadow:4rpx 7rpx 8rpx 0rpx rgba(155,155,155,0.82), 0rpx 8rpx 8rpx 0rpx rgba(155,155,155,0.45);
  border-radius:10rpx;
}
.play_box_center .head_img{
  position: absolute;
  width:120rpx;
  height:120rpx;
  border-radius:50%;
  top:-10%;
  left:40%;
}
.play_box_center .head_img img {
  width:120rpx;
  height:120rpx;
  border-radius:50%;
}
.choose_item{
  display: flex;
  align-items: center;
  padding-top: 15%;
  margin: 0 60rpx 22rpx 31rpx;
}
.choose_item .item_title{
  font-size:28rpx;
  font-weight:400;
  color:rgba(0,0,0,1);
  width: 35%;
}
.choose_item .item_type{
  width:60%;
  height:68rpx;
}
.choose_item .item_type img{
  width:68rpx;
  height:68rpx;
  background:rgba(255,67,71,1);
}
.next_step_choose{
  width:2%;
  height:28rpx;
  color:#999999;
  display:flex;
  text-align: right;
  align-items:center;
}
.choose_tips{
  margin-left: 30rpx;
  font-size:22rpx;
  font-weight:400;
  color:rgba(153,153,153,1);
}
.edit_item{
  display: flex;
  align-items: center;
  margin: 17rpx 60rpx 33rpx 31rpx;
}
/* .publish_squre{
  justify-content: space-around;
} */
.edit_item .item_title{
  font-size:28rpx;
  font-weight:400;
  color:rgba(0,0,0,1);
  width: 35%;
}
.item_input{
  width:60%;
}
/* .picker_box{
  display: flex;
  width: 100%;
  align-items: center;
} */
.item_publish{
  display: flex;
  width:70%;
  justify-content: flex-end
}
.item_input .place-holder{
  font-size:28rpx;
  font-weight:400;
  color:rgba(153,153,153,1);
}
.picker_level, .item_input input{
  font-size:28rpx;
  color: #999;
  padding-bottom: 10rpx;
  border-bottom: 2rpx solid #f4f4f4
}
.edit_item .next_step{
  font-size:28rpx;
  font-weight:400;
  color:rgba(0,0,0,1);
}
.fee_info{
  padding-bottom: 41rpx;
  margin-left: 31rpx;
  font-size:22rpx;
  font-weight:400;
  color:rgba(153,153,153,1);
}
.save_as_package{
  margin: 70rpx 60rpx 0 60rpx;
  height:88rpx;
  background:rgba(255,67,71,1);
  border-radius:4rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size:28rpx;
  font-weight:400;
  color:rgba(255,255,255,1);
}
</style>
