<template lang="html">

  <div class="goods">
    <div class="menu-wrapper" ref="menuWrapper">
      <ul>
        <li v-for="(item,index) in goods" @click="menuClick(index,$event)" :class="index==menuCurrentIndex?'menu-item-selected':'menu-item'">
          <span class="text">
          <!-- 菜单列表前面的打折或者特色或者热销图标使用的一个一个子组件 -->
            <iconMap v-show="item.type>0" :iconType="item.type"></iconMap>
            {{item.name}}
          </span>
        </li>
      </ul>
    </div>
    <div class="foods-wrapper" id="wrapper" ref="foodsWrapper">
      <ul>
        <li v-for="item in goods" class="food-list food-list-hook">
          <h1>{{item.name}}</h1>
          <ul>
            <li v-for="food in item.foods" class="food-item" @click="goDetail(food, $event)">
              <div class="icon">
                <img width="57" height="57" :src="food.icon"/>
              </div>
              <div class="content">
                <h2>{{food.name}}</h2>
                <p class="description" v-show="food.description">{{food.description}}</p>
                <div class="sell-info">
                  <span class="sellCount">月售{{food.sellCount}}份</span>
                  <span class="rating">好评率{{food.rating}}%</span>
                </div>
                <div class="price">
                  <span class="newPrice"><span class="unit">￥</span>{{food.price}}</span>
                  <span v-show="food.oldPrice" class="oldPrice">￥{{food.oldPrice}}</span>
                </div>
                <div class="cartcontrol-wrapper">
                  <cartcontrol :food="food"></cartcontrol>
                </div>
              </div>
            </li>
          </ul>
        </li>
      </ul>
    </div>
    <shopCart :deliveryPrice="seller.deliveryPrice" :minPrice = "seller.minPrice" :selectFoods="selectFoods"></shopCart>
    <foodDetail :food="selectedFood" v-if="selectedFood" ref="myFood"></foodDetail>
  </div>

</template>

<script>
import iconMap from 'components/iconMap/iconMap'
import BScroll from 'better-scroll'
import shopCart from 'components/shopCart/shopCart'
import cartcontrol from 'components/cartcontrol/cartcontrol'
import foodDetail from 'components/foodDetail/foodDetail'
import axios from 'axios'
import Vue from 'vue'

const ERR_OK = 0
const eventHub = new Vue()
export default {
  props: {
    seller: Object
  },
  created() {
    axios.get('static/data.json').then((res) => {
      this.goods = res.data.goods
      this.$nextTick(() => {
        this._initScroll(); // 初始化scroll
        this._calculateHeight(); // 初始化列表高度列表
        this._calculateMenuHeight();
      })
    });
  },
  data() {
    return {
      goods: [],
      listHeight: [],
      foodsScrollY: 0,
      selectedFood: '',
      listMenuHeight: [], // 菜单栏的列表项的高度数组
      menuClientHeight: '', // 菜单栏的可视区的高度
      menuScrollY: 0
    }
  },
  computed: {
    menuCurrentIndex() {
      for (let i = 0, l = this.listHeight.length; i < l; i++) {
        let topHeight = this.listHeight[i]
        let bottomHeight = this.listHeight[i + 1]
        if (!bottomHeight || (this.foodsScrollY >= topHeight && this.foodsScrollY < bottomHeight)) {
          return i
        }
      }
      return 0
    },
    selectFoods() {
      let foods = []
      this.goods.forEach((good) => {
        good.foods.forEach((food) => {
          if (food.count) {
            foods.push(food)
          }
        })
      })
      return foods
    }
  },
  methods: {
    // 初始化scroll
    _initScroll() {
      this.menuScroll = new BScroll(this.$refs.menuWrapper, {
        click: true
      });
      this.foodsScroll = new BScroll(this.$refs.foodsWrapper, {
        click: true,
        probeType: 3 // 设置这个之后，手指不松动的滑动时，比较卡顿
      });
      // 监控滚动事件
      let _self = this
      this.foodsScroll.on('scroll', (pos) => {
        this.foodsScrollY = Math.abs(Math.round(pos.y))
        // 如果当前的菜单选中项的top值大于 菜单的可视区的高度 需要滚动
        let menuScHeight = 10
        let len = this.listMenuHeight.length - 1
        let menuBottom = this.listMenuHeight[this.menuCurrentIndex < len ? (this.menuCurrentIndex + 1) : len]
        if (this.menuClientHeight < menuBottom - this.menuScrollY) {
          this.menuScrollY += menuBottom - this.menuScrollY - this.menuClientHeight
          this.menuScroll.scrollTo(0, -this.menuScrollY, 500)
        } else if (this.menuScrollY > this.listMenuHeight[this.menuCurrentIndex]) {
          this.menuScrollY = this.listMenuHeight[this.menuCurrentIndex]
          this.menuScroll.scrollTo(0, -this.menuScrollY, 500)
        }
      })
    },
    // 初始化食品列表的高度数组
    _calculateHeight() {
      let foodList = this.$refs.foodsWrapper.querySelectorAll('.food-list-hook')
      let height = 0
      this.listHeight.push(height)
      for (let i = 0, l = foodList.length; i < l; i++) {
        let item = foodList[i]
        height += item.clientHeight// 或者使用offsetHeight  在没有滚动条的时候
        // 在有滚动条的时候，获取最大的高度的就是 scrollHeight
        this.listHeight.push(height)
        // 数组的每个数值存储的都是每项食品列表距离食品容器的头部的距离（都是由每项食品列表的高度转化来的）
      }
    },
    // 初始化菜单的列表高度
    _calculateMenuHeight() {
      this.menuClientHeight = this.$refs.menuWrapper.clientHeight
      let menuList = this.$refs.menuWrapper.querySelectorAll('li')
      let menuHeight = 0
      this.listMenuHeight.push(menuHeight)
      for (let i = 0, l = menuList.length; i < l; i++) {
        let item = menuList[i]
        menuHeight += item.clientHeight
        this.listMenuHeight.push(menuHeight)
      }
      // console.log(this.listMenuHeight);
    },
    // 根据点击的菜单，得到index从而取高度数组的第一个数据，从而进行相应的滚动条的调整
    menuClick(index, event) {
      // console.log(event);
      if (!event._constructed) {
        return
      }
      this.foodsScroll.scrollTo(0, -this.listHeight[index], 300)
      // 派发滚动   scrollTo(x, y, time, easing) 滚动到某个位置，x,y 代表坐标，time 表示动画时间，easing 表示缓动函数
    },
    // 获取食品详情
    goDetail(food, event) {
      if (!event._constructed) {
        return
      }
      this.selectedFood = food
      this.$nextTick(() => {
        this.$refs.myFood.showToggle()
      })
    }
  },
  components: {
    iconMap,
    shopCart,
    cartcontrol,
    foodDetail
  }
}

</script>

<style lang="stylus">
@import '../../common/stylus/mixin'
  .goods
    display flex
    position absolute
    top 174px
    bottom 46px
    width 100%
    overflow hidden
    .menu-wrapper
      flex 0 0 80px
      width 80px
      background #f3f5f7
      margin-top: 2px;
      .menu-item-selected
        background white
        font-weight 700
        margin-top -1px
      .menu-item,.menu-item-selected
        position relative
        display table
        height 54px
        line-height 14px
        width 56px
        padding 0 12px
        &:last-child:after
          content none
      .menu-item:after
          position: absolute
          content: ''
          left: 12px
          width: 56px
          bottom: 0
          border-bottom: 1px solid rgba(7,17,27,0.1)
        .text
          display table-cell
          vertical-align middle
          font-size 12px
          font-weight 200
          white-space normal
          line-height 14px
          .iconMap
            vertical-align middle
    .foods-wrapper
      flex 1
      margin-top: 2px;
      .food-list
        h1
          height 26px
          line-height 26px
          padding-left 12px
          font-size 12px
          color rgb(147,153,159)
          background #f3f5f7
          border-left 2px solid #d9dde1
      .food-item
        position relative
        display flex
        margin: 0 18px;
        padding: 18px 0;
        border-bottom 1px solid rgba(7,17,27,0.1)
        .icon
          flex 0 0 57px
        &:last-child
          border-bottom none
        .content
          flex 1
          padding-left 10px
          h2
            margin 2px 0 8px 0
            font-size 14px
            line-height 14px
            height 14px
            font-weight 700
            color rgb(7,17,27)
          .sell-info,.description
            font-size 10px
            color rgb(147,153,159)
            line-height 10px
            .sellCount
              margin-right 4px
          .description
            font-size 10px
            margin-bottom 8px
            line-height: 12px
          .price
            font-size 10px
            font-weight 700
            line-height 24px
            .newPrice
              font-size 14px
              color rgb(240,20,20)
              .unit
                font-size 10px
                font-weight normal
            .oldPrice
              text-decoration line-through
              color rgb(147,153,159)
              padding-left 4px
          .cartcontrol-wrapper
            position: absolute
            right: 0
            bottom 12px
            z-index 20

</style>
