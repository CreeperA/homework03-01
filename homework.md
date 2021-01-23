1.不是响应式数据,需要调用this.$set或Vue.set才行,因为Vue会把一个普通的 JavaScript 对象传入 Vue 实例作为 data 选项,Vue 将遍历此对象所有的 property,并使用 Object.defineProperty 把这些 property 全部转为 getter/setter,用this.$set或Vue.set相当于调用setter方法.

2.diff 的过程就是调用名为 patch 的函数,比较新旧节点,一边比较一边给真实的 DOM 打补丁.

3.

let _Vue = null
export default class VueRouter {
    static install(Vue) {
        // 1、判断当前插件是否已经安装
        if (VueRouter.install.installed) {
            return
        }
        VueRouter.install.installed = true
            // 2、把 vue 构造函数记录到全局变量
        _Vue = Vue
            // 3、把创建 vue 实例时候传入的 router 对象注入到 vue 实例上
            // 混入
        _Vue.mixin({
            beforeCreate() {
                if (this.$options.router) {
                    _Vue.prototype.$router = this.$options.router
                    this.$options.router.init()
                }
            }
        })
    }
    constructor(options) {
        this.options = options
        this.routeMap = {}
        this.data = _Vue.observable({
            current: '/'
        })
    }
    init() {
        this.createRouteMap()
        this.initComponents(_Vue)
        this.initEvent()
    }
    createRouteMap() {
        // 遍历所有的路由规则，把路由规则解析成键值对的形式，存储到 routeMap 中
        this.options.routes.forEach(route => {
            this.routeMap[route.path] = route.component
        })
    }
    initComponents(Vue) {
        const self = this
        Vue.component(
            'router-link', {
                props: {
                    to: String
                },
                render(h) {
                    return h('a', {
                        attrs: {
                            href: '#' + this.to
                        },
                        on: {
                            click: this.clickHandler
                        }
                    }, [this.$slots.default])
                },
                methods: {
                    clickHandler(e) {
                        window.location.hash = '#' + this.to
                        this.$router.data.current = this.to
                        e.preventDefault()
                    }
                }
                // template: '<a :href="to"><slot></slot></a>'
            }
        )

        Vue.component('router-view', {
            render(h) {
                const conmponent = self.routeMap[self.data.current]
                return h(conmponent)
            }
        })
    }

    initEvent() {
        window.addEventListener('load', this.hashChange.bind(this))
        window.addEventListener('hashchange', this.hashChange.bind(this))
    }
    hashChange() {
        if (!window.location.hash) {
            window.location.hash = '#/'
        }
        this.data.current = window.location.hash.substr(1)

    }
}

4.
class Compiler {
    constructor(vm) {
            this.el = vm.$el
            this.vm = vm
            this.compile(this.el)
        }
        // 编译模板，处理文本节点和元素节点
    compile(el) {
            let childNodes = el.childNodes
            Array.from(childNodes).forEach(node => {
                if (this.isTextNode(node)) {
                    // 处理文本节点
                    this.compileText(node)
                } else if (this.isElementNode(node)) {
                    // 处理元素节点
                    this.compileElement(node)
                }
                // 判断 node 节点，是否有子节点，如果有子节点，要递归调用 compile
                if (node.childNodes && node.childNodes.length) {
                    this.compile(node)
                }
            })
        }
        // 编译元素节点，处理指令
    compileElement(node) {
            // 遍历所有的属性节点
            Array.from(node.attributes).forEach(attr => {
                // 判断是否是指令
                let attrName = attr.name
                if (this.isDirective(attrName)) {
                    // v-text --> text
                    attrName = attrName.substr(2)
                    let key = attr.value
                    if (attrName.startsWith('on')) {
                        const event = attrName.replace('on:', '') // 获取事件名
                            // 事件更新
                        return this.eventUpdate(node, key, event)
                    }
                    this.update(node, key, attrName)
                }
            })
        }
        // 编译文本节点，处理插值表达式
    compileText(node) {
        let reg = /\{\{(.+?)\}\}/
        let value = node.textContent
        if (reg.test(value)) {
            let key = RegExp.$1.trim()
            node.textContent = value.replace(reg, this.vm[key])
                // 创建 watcher 对象，当数据改变时更新视图
            new Watcher(this.vm, key, (newValue) => {
                node.textContent = newValue
            })
        }

    }
    update(node, key, attrName) {
        let updateFn = this[attrName + 'Updater']
        updateFn && updateFn.call(this, node, this.vm[key], key)
    }
    eventUpdate(node, key, event) {
        this.onUpdater(node, key, event)
    }


    // 处理 v-text 指令
    textUpdater(node, value, key) {
            node.textContent = value
            new Watcher(this.vm, key, (newValue) => {
                node.textContent = newValue
            })
        }
        // 处理 v-html 指令
    htmlUpdater(node, value, key) {
            node.innerHTML = value
            new Watcher(this.vm, key, (newValue) => {
                node.innerHTML = newValue
            })
        }
        // 处理 v-model 指令
    modelUpdater(node, value, key) {
            node.value = value
            new Watcher(this.vm, key, (newValue) => {
                    node.value = newValue
                })
                // 双向绑定
            node.addEventListener('input', () => {
                this.vm[key] = node.value
            })
        }
        // 处理 v-on 指令
    onUpdater(node, key, event) {
        node.addEventListener(event, (e) => this.vm[key](e))
    }



    // 判断元素属性是否是指令
    isDirective(attrName) {
            return attrName.startsWith('v-')
        }
        // 判断节点是否是文本节点
    isTextNode(node) {
            return node.nodeType === 3
        }
        // 判断节点是否是元素节点
    isElementNode(node) {
        return node.nodeType === 1
    }
}

5.再让我想想