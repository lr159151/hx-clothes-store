<template>
 <div id="detail">
    <detail-nav-bar class="detail-nav" @titleClick="titleClick" ref="nav"/>
    <scroll class="content"
            ref="scroll"
            :probe-type="3"
            @scroll="contentScroll">
      <detail-swiper :top-images="topImages"/>
      <detail-base-info :goods="goods"/>
      <detail-shop-info :shop="shop"/>
      <detail-goods-info :detail-info="detailInfo" @detailImageLoad="detailImageLoad"/>
      <detail-param-info :param-info="paramInfo" ref="params"/>
      <detail-comment-info :comment-info="commentInfo" ref="comment"/>
      <goods-list :goods="recommends" ref="recommend"/>
    </scroll>
    <detail-bottom-bar @addCart="addToCart"/>
    <back-top @click="backClick" v-show="isShowBackTop"/>
 </div>
</template>

<script>
import Scroll from 'components/common/scroll/Scroll'
import GoodsList from 'components/content/goods/GoodsList'

import DetailNavBar from './childComps/DetailNavBar'
import DetailSwiper from './childComps/DetailSwiper'
import DetailBaseInfo from './childComps/DetailBaseInfo'
import DetailShopInfo from './childComps/DetailShopInfo'
import DetailGoodsInfo from './childComps/DetailGoodsInfo'
import DetailParamInfo from './childComps/DetailParamInfo'
import DetailCommentInfo from './childComps/DetailCommentInfo'
import DetailBottomBar from './childComps/DetailBottomBar'

import { getDetail, getRecommend, Goods, Shop, GoodsParam } from 'network/detail'
import { debounce } from 'common/utils'
import { backTopMixin } from 'common/mixin'
import { mapActions } from 'vuex'

  export default {
    name: 'Detail',
    components: {
      Scroll,
      GoodsList,
      DetailNavBar,
      DetailSwiper,
      DetailBaseInfo,
      DetailShopInfo,
      DetailGoodsInfo,
      DetailParamInfo,
      DetailCommentInfo,
      DetailBottomBar
    },
    mixins: [backTopMixin],
    data() {
      return {
        iid: null,
        topImages: [],
        goods: {},
        shop: {},
        detailInfo: {},
        paramInfo: {},
        commentInfo: {},
        recommends: [],
        themeTopYs: [],
        getThemeTopY: null,
        currentIndex: 0
      }
    },
    created() {
      // this.$route.params??????iid???????????????????????????/:iid
      // 1.???????????????iid
      this.iid = this.$route.params.iid

      // 2.??????iid??????????????????
      getDetail(this.iid).then(res => {
        console.log(res);

        const data = res.result
        // 1.???????????????????????????
        this.topImages = data.itemInfo.topImages

        // 2.??????????????????
        this.goods = new Goods(data.itemInfo, data.columns, data.shopInfo.services)

        // 3.????????????????????????
        this.shop = new Shop(data.shopInfo)

        // 4.????????????????????????
        this.detailInfo = data.detailInfo

        // 5.?????????????????????
        this.paramInfo = new GoodsParam(data.itemParams.info, data.itemParams.rule)

        // 6.???????????????????????????
        if(data.rate.cRate !== 0){
          this.commentInfo = data.rate.list[0]
        }
      })

      // 3.??????????????????
      getRecommend().then(res => {
        this.recommends = res.data.list
      })

      // 4.??????getThemeTopY
      this.getThemeTopY = debounce(() => {
        this.themeTopYs = []
        this.themeTopYs.push(0)
        this.themeTopYs.push(this.$refs.params.$el.offsetTop)
        this.themeTopYs.push(this.$refs.comment.$el.offsetTop)
        this.themeTopYs.push(this.$refs.recommend.$el.offsetTop)
        this.themeTopYs.push(Number.MAX_VALUE)
        // console.log(this.themeTopYs);
      }, 50)
    },
    methods: {
      ...mapActions(['addCart']),
      titleClick(index) {
        this.$refs.scroll.scrollTo(0, (-this.themeTopYs[index]), 100)
      },
      detailImageLoad() {
        debounce(this.$refs.scroll.refresh, 100)
        this.getThemeTopY()
      },
      contentScroll(position) {
        // 1.??????y???
        const positionY = -position.y
        // 2.positionY???????????????????????????
        // [0, 7938, 9120, 9452, Numbre.MAX_VALUE]
        // console.log(Numbre.MAX_VALUE);
        // 
        // positionY ??? 0 ??? 7938 ??????, index = 0
        // positionY ??? 7938 ??? 9120 ??????, index = 1
        // positionY ??? 9120 ??? 9452 ??????, index = 2
        // positionY ?????? 9452 ???, index = 3
        // Better-Scroll v2.4.2????????????Numbre.MAX_VALUE????????????bug????????????nav????????????????????????

        let length = this.themeTopYs.length
        for(let i = 0; i < length; i++) {
          // if (this.currentIndex !== i && ((i < length - 1 && positionY >= this.themeTopYs[i] && positionY < this
          //     .themeTopYs[i+1]) || (i === length - 1 && positionY >= this.themeTopYs[i]))) {

          if (this.currentIndex !== i && (positionY >= this.themeTopYs[i] && positionY < this.themeTopYs[i+1])) {
            this.currentIndex = i
            this.$refs.nav.currentIndex = this.currentIndex
          }
        }
        // 3.????????????????????????
        this.isShowBackTop = (-position.y) > 1000
      },
      addToCart() {
        // 1.????????????????????????????????????
        const product = {}
        product.image = this.topImages[0]
        product.title = this.goods.title
        product.desc = this.goods.desc
        product.price = this.goods.realPrice
        product.iid = this.iid

        // 2.??????????????????????????????(1.Promise 2.mapActions)
        // this.$store.carList.push(product)
        // this.$store.commit('addCart', product)
        this.addCart(product).then(res => {
          // console.log(res);

          this.$toast.show(res)
          // console.log(this.$toast);
        })
        // ??????Vuex???actions
        // this.$store.dispatch('addCart', product).then(res => {
        //   console.log(res);
        // })
      }
    }
  }

</script>
<style scoped>
#detail {
    height: 100vh;
    position: relative;
    z-index: 1;
    /* overflow: auto; */
    background-color: #fff;
    /* ??????BetterScroll????????????????????????bug */
    overflow: hidden;
    padding-top: 44px;
  }

  .content {
    height: calc(100% - 58px - 44px);
  }
  .detail-nav {
    position: fixed;
    left: 0;
    right: 0;
    top: 0;
    z-index: 9;
    background-color: #fff;
  }
  .back-top {
    position: fixed;
    right: 10px;
    bottom: 65px;
  }
</style>
