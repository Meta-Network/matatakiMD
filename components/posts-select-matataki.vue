<template>
  <div v-loading="selectLoading" class="select">
    <v-select
      v-model="value"
      label="label"
      :filterable="false"
      :options="selectList"
      placeholder="请选择"
      @open="onOpen"
      @close="onClose"
      @search="query => search = query"
    >
      <template #list-footer>
        <li v-show="hasNextPage" ref="load" class="loader">
          Loading more...
        </li>
      </template>
    </v-select>
  </div>
</template>

<script lang="ts">
import { isEmpty } from 'lodash'
import {
  Component,
  Vue,
  Prop,
  Watch
} from 'nuxt-property-decorator'
import vSelect from 'vue-select'
import 'vue-select/dist/vue-select.css'
import { getPostsTimeRanking } from '../api/index'
import { PostsTimeRankingDataProps, PostsTimeRankingDataListProps, userProps } from '../types/index.d'

interface valueProps extends PostsTimeRankingDataListProps {
  label: string
}

@Component({
  components: {
    'v-select': vSelect
  }
})
export default class PostsSelectMatataki extends Vue {
  @Prop({
    type: Object,
    required: true
  })
  readonly usersData!: userProps

  @Prop({
    type: Object,
    required: true
  })
  readonly form!: PostsTimeRankingDataListProps

  // select value
  value: valueProps = { } as valueProps

  // posts data
  postsData: PostsTimeRankingDataProps = {
    count: 0,
    list: []
  }

  // select loading
  selectLoading: boolean = false

  observer: any = null
  page: number = 1
  search: string = ''

  mounted () {
    if (process.client) {
      this.init()
    }
  }

  @Watch('value')
  onWatchChanged (val: PostsTimeRankingDataListProps) {
    console.log('val', val)
    this.$emit('changedValue', val)
  }

  // 是否有下一页
  get hasNextPage (): boolean {
    if (this.search) {
      return false
    }
    return this.postsData.list.length < this.postsData.count
  }

  // 获取下拉列表
  get selectList (): PostsTimeRankingDataListProps[] {
    if (this.search) {
      // 因为调用查询接口毕竟慢而且不太好用 干脆数组内搜索 有需要再修改
      return this.postsData.list.filter(i => i.title.includes(this.search))
    } else {
      return this.postsData.list
    }
  }

  async init () :Promise<void> {
    await this.postsTimeRanking()

    /**
     * You could do this directly in data(), but since these docs
     * are server side rendered, IntersectionObserver doesn't exist
     * in that environment, so we need to do it in mounted() instead.
     */
    this.observer = new IntersectionObserver(this.infiniteScroll)

    await this.setValueFn()
  }

  // 获取 MTK 文章列表
  async postsTimeRanking (): Promise<void> {
    this.selectLoading = true

    try {
      const res = await getPostsTimeRanking({
        author: this.usersData.id,
        page: this.page,
        pagesize: 20
      })
      if (res.code === 0) {
        const data: PostsTimeRankingDataProps = res.data
        this.postsData.count = data.count
        // copy title, 因为 title key 和 父组件 title 离奇冲突，所以拷贝一个 label 使用
        const list = data.list.map(i => ({ ...i, label: i.title }))
        this.postsData.list = this.postsData.list.concat(list)
      } else {
        throw new Error(res.message)
      }
    } catch (e) {
      console.log(e.toString())
    } finally {
      this.selectLoading = false
    }
  }

  // select open
  async onOpen (): Promise<void> {
    if (this.hasNextPage) {
      await this.$nextTick()
      this.observer.observe(this.$refs.load)
    }
  }

  // select close
  onClose (): void {
    this.observer.disconnect()
  }

  // 无限滚动
  async infiniteScroll ([{ isIntersecting, target }]: any): Promise<void> {
    if (isIntersecting) {
      const ul = target.offsetParent
      const scrollTop = target.offsetParent.scrollTop
      this.page += 1
      await this.postsTimeRanking()
      await this.$nextTick()
      ul.scrollTop = scrollTop
    }
  }

  // 设置 value
  setValueFn () {
    if (!isEmpty(this.form)) {
      if (this.form && this.form.id && this.form.title) {
      // 处理 alias
        this.value = {
          ...this.form,
          label: this.form.title
        }
      }
    }
  }
}
</script>

<style lang="less" scoped>
.loader {
  text-align: center;
  color: #bbbbbb;
}
.select /deep/ .vs__dropdown-toggle{
  border: 1px solid #DCDFE6;
  padding: 6px 0 10px;
}
.select /deep/ .vs__search,
.select /deep/ .vs__search:focus {
  color: #606266;
}
</style>
