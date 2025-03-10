<template>
  <div
    v-scroll="handleLoadMore"
    class="aile-autocomplete"
  >
    <el-autocomplete
      ref="autocomplete"
      v-bind="mergeAttrs"
      :fetch-suggestions="handleFetchSuggestions"
      :value-key="valueKey"
      :style="calcStyle"

      @clear="handleClear"
      @select="handleSelect"
      @change="handleChange"
    >
      <template
        v-if="$slots.prepend"
        #prepend
      >
        <slot name="prepend" />
      </template>
      <template
        v-if="$slots.append"
        #append
      >
        <slot name="append" />
      </template>
      <template
        v-if="$slots.prefix"
        #prefix
      >
        <slot name="prefix" />
      </template>
      <template
        v-if="$slots.suffix"
        #suffix
      >
        <slot name="suffix" />
      </template>
      <template #default="{ item }">
        <slot :item="item" />
      </template>
    </el-autocomplete>
  </div>
</template>

<script>
import { DefaultConfig, DefaultAutocompleteAttrs } from './config'

export default {
  name: 'AileAutocomplete',
  directives: {
    scroll: {
      mounted (el, binding) {
        const child = el.querySelector('.el-autocomplete')
        const id = child.getAttribute('ariadescribedby')
        const poper = document.getElementById(id)
        const SELECTDOWN_DOM = poper.querySelector('.el-autocomplete-suggestion .el-autocomplete-suggestion__wrap')
        // 这里不能使用箭头函数！
        SELECTDOWN_DOM.addEventListener('scroll', function () {
          /**
           * scrollHeight 获取元素内容高度(只读)
           * scrollTop 获取或者设置元素的偏移值,
           *  常用于:计算滚动条的位置, 当一个元素的容器没有产生垂直方向的滚动条, 那它的scrollTop的值默认为0.
           * clientHeight 读取元素的可见高度(只读)
           * 如果元素滚动到底, 下面等式返回true, 没有则返回false:
           * ele.scrollHeight - ele.scrollTop === ele.clientHeight;
           */
          const CONDITION = this.scrollHeight - this.scrollTop <= this.clientHeight
          if (CONDITION) {
            binding.value()
          }
        })
      }
    }
  },
  inheritAttrs: false,
  props: {
    config: {
      type: Object,
      default: () => ({})
    }
  },
  data () {
    return {
      callback: null,
      respSource: null,
      currentPage: 1,
      total: 0,
      queryString: '',
      clearClick: false,
      loadMoreDisabled: false
    }
  },
  computed: {
    mergeConfig () {
      let { data } = this.config
      if (data && data.length) {
        const dataItem = data[0]
        if (typeof dataItem !== 'object') {
          data = data.map(item => ({ [this.mergeAttrs.valueKey]: item }))
        } else if (!{}.hasOwnProperty.call(dataItem, this.mergeAttrs.valueKey)) {
          throw Error(
            `Each item of data should has ${this.mergeAttrs.valueKey} property`
          )
        }
      } else {
        data = []
      }
      return {
        ...DefaultConfig,
        ...this.$aileAutocomplete.config,
        ...this.config,
        data
      }
    },

    mergeAttrs () {
      return {
        ...DefaultAutocompleteAttrs,
        ...this.$aileAutocomplete.attrs,
        ...this.$attrs
      }
    },

    calcStyle () {
      const style = {}
      if (this.mergeConfig.width) style.width = this.mergeConfig.width
      return style
    },

    /**
     * 是否可以继续滚动加载
     */
    canLoadMore () {
      return (
        this.mergeConfig.scrollable &&
        this.mergeConfig.remoteMethod &&
        this.currentPage * this.mergeConfig.pageSize < this.total
      )
    }
  },
  methods: {
    handleChange (val) {
      if (this.mergeConfig.lazyTrim) {
        val = val.trim()
        this.$emit('update:modelValue', val)
      }
      this.$emit('change', val)
    },
    handleSelect (item) {
      this.loadMoreDisabled = true
      this.$emit('select', item)
    },
    focus () {
      this.$refs.input.focus()
    },
    handleClear () {
      this.clearClick = true
      this.$emit('clear')
    },

    /**
     * 远程获取数据
     */
    getRemoteList (callback) {
      const { scrollable } = this.mergeConfig
      this.loadMoreDisabled = false
      if (scrollable) {
        // 滚动分页加载
        this.currentPage = 1
        this.callback = callback
        return this.fetchListByScroll(callback)
      }
      // 无分页加载
      return this.fetchListOnce(callback)
    },

    /**
     * 采用无分页加载
     */
    async fetchListOnce () {
      const {
        queryField,
        remoteMethod,
        requestParams,
        respFormatter,
        respDataField
      } = this.mergeConfig
      const query = { [queryField]: this.queryString, ...requestParams }
      const res = await remoteMethod(query)
      return respFormatter(res[respDataField])
    },

    /**
     * 采用分页加载
     */
    async fetchListByScroll () {
      const {
        queryField,
        remoteMethod,
        requestParams,
        respFormatter,
        pageField,
        sizeField,
        pageSize,
        respDataField,
        respTotalField
      } = this.mergeConfig
      const query = { [queryField]: this.queryString, ...requestParams }
      query[pageField] = this.currentPage
      query[sizeField] = pageSize
      try {
        const res = await remoteMethod(query)
        const respSource = respFormatter(res[respDataField])
        this.total = res[respTotalField]
        if (this.currentPage === 1) {
          this.respSource = respSource
          return respSource
        }
        this.respSource = [...this.respSource, ...respSource]

        this.callback(this.respSource)
      } catch (error) {
        this._reset()
      }
    },

    /**
     * 滚动加载
     */
    handleLoadMore () {
      if (this.loadMoreDisabled || !this.canLoadMore) return
      this.currentPage++
      this.fetchListByScroll()
    },

    /**
     * 获取输入建议数据
     */
    async handleFetchSuggestions (queryString, cb) {
      if (this.clearClick) {
        // hack 当点击清除按钮 防止请求
        this.clearClick = false
        return cb([])
      }
      this.queryString = typeof queryString === 'object' ? '' : queryString
      const { remote, data } = this.mergeConfig
      let results = []
      if (remote) {
        // 返回远程请求获取数据
        results = await this.getRemoteList(cb)
      } else {
        // 返回传入的静态数据
        results = queryString
          ? data.filter(item => (item[this.mergeAttrs.valueKey] || '').includes(queryString)
          )
          : data
      }
      cb(results)
    },

    /**
     * 重置
     */
    _reset () {
      this.respSource = []
      this.currentPage = 1
      this.total = 0
      this.queryString = ''
      this.clearClick = false
    }
  }
}
</script>

<style scoped>
.aile-autocomplete {
  display: inline-block;
  width: auto;
}
</style>
