<template>
  <div>
    {{loginStatus}}
    <div style="width:1060px;height:1170px;overflow:auto">
      <div style="display: table;border:1px solid gray;" v-for="(cell,i) in codeDisplay" :key="i">
        <span v-if="cell.sequence !== ''" style="display: table-cell; vertical-align: middle; color: red;">[{{cell.sequence}}]</span>
        <span v-else></span>
        <div style="width:1010px; display: inline-block;border:1px solid LightSteelBlue;" v-html="cell.text"></div>
      </div>
    </div>
    <div>
        <div>
          <select id="language">
            <option value="python3">python3</option>
          </select>
          <button id="send" @click="send">运行</button>
          <button id="tab" @click="tab">补全</button>
        </div>
        <div id="input" v-show="isInput">
          <input type="text" v-model="inputData" @change="scanf">
        </div>
        <div>
          <div id="container"></div>
        </div>
    </div>
  </div>
</template>

<script>
import Cookies from 'js-cookie'
import axios from 'axios'
import qs from 'qs'
import marked from 'marked'
import * as monaco from 'monaco-editor'
export default {
  name: 'Index',
  data () {
    return {
      httpUrl: 'http://localhost:8090/',
      wsUrl: 'ws://localhost:8090/',
      loginStatus: '未登录',
      cookieExpires: 30,
      wsChat: null,
      wsInput: null,
      editor: null,
      inputData: '',
      isInput: false,
      codeDisplay: []
    }
  },
  methods: {
    login () {
      // 登录
      axios.post(`${this.httpUrl}api/login`, qs.stringify({
        name: '1',
        passwd: '1'
      })).then((res) => {
        console.log(res)
        this.loginStatus = res.data.token
        Cookies.set('token', res.data.token, {expires: this.cookieExpires || 1})
      }).catch((err) => {
        console.log(err)
      })
    },
    initEditor () {
      // 初始化编辑器，确保dom已经渲染
      // value 编辑器初始显示文字
      // language 语言支持自行查阅demo
      // automaticLayout 自动布局
      // theme 官方自带三种主题vs, hc-black, or vs-dark
      this.editor = monaco.editor.create(document.getElementById('container'), {
        value: '',
        language: 'python',
        automaticLayout: true,
        theme: 'vs'
      })
    },
    getValue () {
      // 获取编辑器中的文本
      return this.editor.getValue()
    },
    setValue (content) {
      this.editor.setValue(content)
    },
    send () {
      let s = Boolean(this.getValue())
      if (s) {
        this.runCode('compute', this.getValue(), {})
        this.setValue('')
      }
    },
    tab () {
      let s = Boolean(this.getValue())
      if (s) {
        this.runCode('complete', this.getValue(), {})
        this.setValue('')
      }
    },
    scanf () {
      let s = Boolean(this.inputData)
      if (s) {
        this.wsInput.send(this.inputData)
        this.inputData = ''
        this.isInput = false
      }
    },
    runCode (type, data, desc) {
      this.wsChat.send(JSON.stringify({'mode': 'edit', 'language': 'python3', 'type': type, 'data': data, 'desc': desc}))
    },
    onopen () {
      console.log('socket连接成功')
    },
    onerror () {
      console.log('连接错误')
    },
    onmessage (e) {
      let res = JSON.parse(e.data)
      console.log(res)
      if (res.type === 'code') {
        this.codeDisplay.push(
          {
            'id': res.id,
            'sequence': '*',
            'text': res.data.replace(/[<">']/g, (a) => { return {'<': '&lt;', '"': '&quot;', '>': '&gt;', "'": '&#39;'}[a] }).replace(/\n/g, '<br/>').replace(/\s/g, '&nbsp;')
          }
        )
      } else if (res.type === 'sequence') {
        let cell = this.codeDisplay.find(cell => cell.id === res.id)
        let s = Boolean(cell)
        if (s) {
          cell['sequence'] = res.data
        }
      } else if (res.type === 'input') {
        this.isInput = true
      } else if (res.type === 'output') {
        if (res.data.type === 'text') {
          this.codeDisplay.push(
            {
              'id': res.id,
              'sequence': 'Out',
              'text': res.data.content.replace(/[<">']/g, (a) => { return {'<': '&lt;', '"': '&quot;', '>': '&gt;', "'": '&#39;'}[a] }).replace(/\n/g, '<br/>').replace(/\s/g, '&nbsp;')
            }
          )
        } else if (res.data.type === 'image') {
          this.codeDisplay.push(
            {
              'id': res.id,
              'sequence': 'Out',
              'text': '<img src="data:image/png;base64,' + res.data.content + '&#10;" alt="">'
            }
          )
        } else if (res.data.type === 'markdown') {
          this.codeDisplay.push(
            {
              'id': res.id,
              'sequence': 'Out',
              'text': marked(res.data.content)
            }
          )
        } else if (res.data.type === 'html') {
          this.codeDisplay.push(
            {
              'id': res.id,
              'sequence': 'Out',
              'text': res.data.content
            }
          )
        }
      } else if (res.type === 'stream') {
        this.codeDisplay.push(
          {
            'id': res.id,
            'sequence': '',
            'text': res.data.replace(/[<">']/g, (a) => { return {'<': '&lt;', '"': '&quot;', '>': '&gt;', "'": '&#39;'}[a] }).replace(/\n/g, '<br/>').replace(/\s/g, '&nbsp;')
          }
        )
      } else if (res.type === 'error') {
        this.codeDisplay.push(
          {
            'id': res.id,
            'sequence': '',
            'text': res.data.map(item => item.replace(/[<">']/g, (a) => { return {'<': '&lt;', '"': '&quot;', '>': '&gt;', "'": '&#39;'}[a] }).replace(/\n/g, '<br/>').replace(/\s/g, '&nbsp;')).join('<br/>')
          }
        )
      } else if (res.type === 'complete') {
        this.codeDisplay.push(
          {
            'id': res.id,
            'sequence': 'Out',
            'text': res.data
          }
        )
      } else {
        console.log(`不识别类型:${res.type}`)
      }
    },
    onclose () {
      console.log('socket已经关闭')
    }
  },
  created () {
    if (this.wsChat) {
      this.wsChat.close()
    }
    if (this.wsInput) {
      this.wsInput.close()
    }
    this.wsChat = new WebSocket(`${this.wsUrl}wechat`)
    this.wsInput = new WebSocket(`${this.wsUrl}input`)
    this.wsChat.onopen = this.onopen
    this.wsChat.onerror = this.onerror
    this.wsChat.onmessage = this.onmessage
    this.wsChat.onclose = this.onclose
  },
  mounted () {
    this.initEditor()
  },
  destroyed () {
    if (this.wsInput) {
      this.wsInput.close()
    }
    if (this.wsChat) {
      this.wsChat.close()
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#container{
  width: 100%;
  height: 500px;
}
</style>
