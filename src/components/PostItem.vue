<template>
    <li>
        <div class="content" v-html="display(post.content)"  @click="showImg"></div>
        
        <!-- 帖子的详细信息 -->
        <ul class="info">
            <li>{{ index }}楼</li>
            <li>作者：<router-link :to="'/user/' + post.userId + '/page=1&size=10'">{{ post.nickname }}</router-link></li>
            <li>发帖时间：{{ (s => {const t = s.split(/[+T]/); return t[0] + " " + t[1] })(post.created) }}</li>
            <li v-if="seen(post)">更新时间：{{ (s => {const t = s.split(/[+T]/); return t[0] + " " + t[1] })(post.updated) }}</li>
            <li v-if="check(post.userId) && index !== 1"><span class="a" @click="seenEdit = !seenEdit">{{ seenEdit ? '收起编辑': '编辑' }}</span></li>
            <li v-if="index !== 1"><span class="a" @click.prevent="seenReply = !seenReply">{{ seenReply ? '收起回复': '回复' }}</span></li>
        </ul>
        <transition name="fade"><Reply v-if="seenEdit" @reload="reload" :preContent="post.content" :postid="mainId" :settedReplyId="post.id" :isEdit="true"></Reply></transition>
        <transition name="fade"><Reply v-if="seenReply" @reload="reload" preContent="" :postid="mainId" :settedReplyId="post.id" :isEdit="false"></Reply></transition>
        
        <!-- 楼中楼 -->
        <div v-if="checkReply(displayInfo)" class="sub-reply">
            <ul class="sub-ul">
                <li v-for="(post, index) in displayInfo" :key="index" class="reply-style">
                    <div class="content" v-html="display(find(post.replyId) + post.content)"  @click="showImg"></div>
                    <ul class="info">
                        <li>{{ index + 1 }}楼</li>
                        <li>作者：<router-link :to="'/user/' + post.userId + '/page=1&size=10'">{{ post.nickname }}</router-link></li>
                        <li>发帖时间：{{ (s => {const t = s.split(/[+T]/); return t[0] + " " + t[1] })(post.created) }}</li>
                        <li v-if="seen(post)">更新时间：{{ (s => {const t = s.split(/[+T]/); return t[0] + " " + t[1] })(post.updated) }}</li>
                        <li v-if="check(post.userId)"><span class="a" @click="edit(index)">{{ seenEdits[index] ? '收起编辑': '编辑' }}</span></li>
                        <li><span class="a" @click.prevent="replyit(index)">{{ seenReplys[index] ? '收起回复': '回复' }}</span></li>
                    </ul>
                    <transition name="fade"><Reply v-if="seenEdits[index]" @reload="reload" :preContent="post.content" :postid="mainId" :settedReplyId="post.id" :isEdit="true"></Reply></transition>
                    <transition name="fade"><Reply v-if="seenReplys[index]" @reload="reload" preContent="" :postid="mainId" :settedReplyId="post.id" :isEdit="false"></Reply></transition>
                </li>
            </ul>
        </div>
        <div class="more" v-if="startIndex + step < cnt"><span class="a" @click="more">加载更多</span></div>
    </li>
</template>

<script>
import Reply from '@/components/Reply.vue'
import analyzeEmotion from '@/components/public.js'

export default {
    name: 'PostItem',
    props: ['post', 'index', 'mainId'],
    inject: ['reload'],
    components: {
        Reply
    },
    data(){
        return {
            seenEdit: false,
            seenReply: false,
            displayInfo: [],
            seenEdits: [],
            seenReplys: [],
            startIndex: 0,
            cnt: 0,
            step: 5,
            pos: undefined
        }
    },
    created(){
        if(this.post.reply !== undefined){
            this.cnt = this.post.reply.length
            this.seenEdits.length = this.cnt
            this.seenEdits.fill(false)
            this.seenReplys.length = this.cnt
            this.seenReplys.fill(false)
            if(this.cnt <= this.step){
                    this.displayInfo = this.post.reply
            } else {
                for(let i = 0; i < this.step; i++){
                    this.displayInfo.push(this.post.reply[i])
                }
            }
        }
    },
    methods: {
        display(content){ // 渲染帖子内容
            return '<div class="setSize">' + analyzeEmotion(content) + '</div>'
        },
        seen(post){ // 帖子更新过才显示更新时间
            return post.updated !== post.created
        },
        check(userid){ // 本人的帖子才允许编辑
            return userid === this.$store.state.userid
        },
        checkReply(postReply){ // 存在楼中楼才渲染
            return postReply !== undefined && postReply.length > 0
        },
        edit(index){ // 设置“编辑回帖”状态
            this.$set(this.seenEdits, index, !this.seenEdits[index])
        },
        replyit(index){ // 设置“回复回帖”状态
            this.$set(this.seenReplys, index, !this.seenReplys[index])
        },
        find(replyId){ // 如果是回复楼中楼，还需要注明回复的是几楼
            for(let i = 0, len = this.post.reply.length; i < len; i++){
                if(this.post.reply[i].id === replyId){
                    return '&gt; 回复 ' + this.post.reply[i].nickname + ' 在 ' + (i + 1) + ' 楼的帖子\n\n'
                }
            }
            return ""
        },
        more(){ // 楼中楼，点击加载更多
            this.startIndex += this.step
            if(this.cnt <= this.startIndex + this.step){
                for(let i = this.startIndex; i < this.cnt; i++){
                    this.displayInfo.push(this.post.reply[i])
                }
            } else {
                for(let i = this.startIndex; i < this.startIndex + this.step; i++){
                    this.displayInfo.push(this.post.reply[i])
                }
            }
        },
        showImg(e){ // 对长图添加点击放大，再点击复原的功能
            if(e){
                if(e.target.tagName === 'IMG'){
                    if(e.target.height !== e.target.naturalHeight || e.target.className === 'big'){
                        if(e.target.className === 'big'){
                            e.target.className = ''
                            document.documentElement.scrollTop = this.pos
                        } else {
                            e.target.className = 'big'
                            this.pos = document.documentElement.scrollTop || document.body.scrollTop;
                        }
                    }
                }
            }
        }
    }
}
</script>

<style scoped>
.fade-enter-active, .fade-leave-active {
    transition: opacity .3s linear;
}
.fade-enter, .fade-leave-to {
    opacity: 0;
}

.sub-reply {
    margin-left: 30px;
    margin-bottom: 10px;
    font-size: 14px;
}

.sub-ul {
    list-style-type: none;
    font-size: 0.95em;
}

.reply-style {
    border-top: 1px solid pink;
}
</style>
