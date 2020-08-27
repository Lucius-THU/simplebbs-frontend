<template>
    <div class="favorites">
        <div class="container">
            <ul id="post-list">
                <li v-for="(post, index) in posts" :key="index" @click="getPost(post, $event)">
                    <h3>{{ post.title }}</h3>
                    <div class="content" :class="{ summary: post.content.length > 1000 }" v-html="display(post.content)"></div>
                    <div class="hide-bar" v-if="post.content.length > 1000"></div>
                    <ul class="info">
                        <li>作者：<router-link target="_blank" :to="'/user/' + post.userId + '/page=1&size=10'">{{ post.nickname }}</router-link></li>
                        <li>发帖时间：{{ (s => {const t = s.split(/[+T]/); return t[0] + " " + t[1] })(post.created) }}</li>
                        <li v-if="seen(post)">{{ latestTimeTag(post) }}：{{ (s => {const t = s.split(/[+T]/); return t[0] + " " + t[1] })(latestTime(post)) }}</li>
                    </ul>
                </li>
            </ul>
        </div>
        <NextPage v-if="nextSeen" @click.native="Pageto(1)"></NextPage>
        <PrevPage v-if="prevSeen" @click.native="Pageto(-1)"></PrevPage>
        <ToPage :now="Number(page)" :total="Number(total)" @to="Pageto"></ToPage>
    </div>
</template>

<script>
import NextPage from '@/components/NextPage.vue'
import PrevPage from '@/components/PrevPage.vue'
import ToPage from '@/components/ToPage.vue'
import analyzeEmotion from '@/components/public.js'

export default {
    name: 'Favorites',
    components: {
        NextPage,
        PrevPage,
        ToPage
    },
    data(){
        return {
            posts: [],
            nextSeen: false,
            prevSeen: false,
            page: this.$route.params.page,
            size: this.$route.params.size,
            total: 1
        }
    },
    beforeRouteUpdate(to, from, next) {
        this.page = to.params.page
        this.size = to.params.size
        this.refresh()
        next()
    },
    beforeRouteLeave(to, from, next) {
        if(to.path === '/login'){
            next()
        } else {
            this.page = to.params.page
            this.size = to.params.size
            this.refresh()
            next()
        }
    },
    created(){
        document.title = '收藏 - 清软论坛'
        this.refresh()
    },
    methods: {
        Pageto(t){
            this.$router.push('/favorites/page=' + (Number(this.page) + t) + '&size=' + this.size)
        },
        refresh(){
            if(localStorage.posts === undefined || localStorage.posts === ''){
                this.total = 1
                this.page = 1
            } else {
                this.total =  parseInt((localStorage.posts.match(/;/g).length + Number(this.size) - 1) / this.size)
                if(this.page > this.total){
                    this.page = this.total
                } else if(this.page < 1){
                    this.page = 1
                }
                if(this.page < this.total){
                    this.nextSeen = true
                } else this.nextSeen = false
                if(this.page > 1){
                    this.prevSeen = true
                } else this.prevSeen = false
                this.posts = []
                const matches = localStorage.posts.match(/\[\d+\];/g)
                for(let i = (this.page - 1) * this.size, len = Math.min(matches.length, this.page * this.size); i < len; i++){
                    let word = matches[i].replace(/\[|\]|;/g, '')
                    this.axios.get('/api/v1/post/' + word).then(response => {
                        const info = {}
                        info.id = response.data.id
                        info.userId = response.data.userId
                        info.nickname = response.data.nickname
                        info.title = response.data.title
                        info.content = response.data.content
                        info.created = response.data.created
                        info.updated = response.data.updated
                        info.lastRepliedTime = response.data.lastRepliedTime
                        this.posts.push(info)
                    }).catch(error => {
                        if(error.response.status === 401) this.$router.push('/login')
                    })
                }
            }
        },
        seen(post){
            return post.lastRepliedTime !== post.created
        },
        latestTime(post){
            return post.updated < post.lastRepliedTime ? post.lastRepliedTime: post.updated
        },
        latestTimeTag(post){
            return post.updated < post.lastRepliedTime ? '最新回帖时间': '更新时间'
        },
        getPost(post, event){
            if(event.target.tagName !== 'A'){
                const newPage = this.$router.resolve('/post/' + post.id)
                window.open(newPage.href, '_blank')
            }
        },
        display(content){
            return '<div class="setSize">' + analyzeEmotion(content) + '</div>'
        }
    }
}
</script>

<style scoped>
#post-list > li {
    cursor: pointer;
}

.summary {
    height: 225px;
    overflow: hidden;
    text-overflow: clip;
}

.hide-bar {
    position: absolute;
    width: 98%;
    height: 100px;
    bottom: 60px;
    background: linear-gradient(rgba(255, 255, 255, 0), rgba(255, 255, 255, 1) 70%);
}
</style>