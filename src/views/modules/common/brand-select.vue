<template>
  <div>
    <el-select placeholder="请选择" v-model="brandId" filterable clearable>
      <el-option
        v-for="item in brands"
        :key="item.brandId"
        :label="item.brandName"
        :value="item.brandId"
      ></el-option>
    </el-select>
  </div>
</template>

<script>
export default {
  components: {},
  props: {},
  data () {
    return {
      catId: 0,
      brands: [
        {
          label: 'a',
          value: 1
        }
      ],
      brandId: '',
      subscribe: null
    }
  },
  computed: {},
  watch: {
    brandId (val) {
      this.PubSub.publish('brandId', val)
    }
  },
  methods: {
    getCatBrands () {
      this.$http({
        url: this.$http.adornUrl('/product/categorybrandrelation/brands/list'),
        method: 'get',
        params: this.$http.adornParams({
          catId: this.catId
        })
      }).then(({data}) => {
        this.brands = data.data
      })
    }
  },
  created () {
  },
  mounted () {
    this.subscribe = PubSub.subscribe('catPath', (msg, val) => {
      this.catId = val[val.length - 1]
      this.getCatBrands()
    })
  },
  beforeCreate () {
  },
  beforeMount () {
  },
  beforeUpdate () {
  },
  updated () {
  },
  beforeDestroy () {
    PubSub.unsubscribe(this.subscribe)
  },
  destroyed () {
  },
  activated () {
  }
}
</script>
<style>
</style>
