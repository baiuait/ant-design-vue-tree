# ant-design-vue-tree
// 树数据
treeData: any[]
// 树替换字段
replaceFields: {key: string, title: string, children: string}
// 树当前选择节点keys
v-model:selectedKeys: string[]
// 是否默认展开全部节点(通过监听loading实现 需要配合使用)
defaultExpandAll: boolean
// 是否展示搜索框
showSearch: boolean
// 菜单功能项数组 {name: '菜单名', method: '点击菜单执行的函数', flag 符合什么条件时展示}
operationList: {name: string, method: (record: 当前节点对象) => void, flag: (record: 当前节点对象) => boolean}
// 不需要菜单功能的节点key值, 比如说未分类不需要展示菜单 key为NONE 那noOperationNode的值就是["NONE"]
noOperationNode: string[]
// 拖拽相关对象 method: 拖拽完成后执行的函数
drop: { method: (data: 拖拽完成后的数据, info: DropEvent) => void }
// 是否加载状态 true为加载状态, false为不加载 spin框默认有240的高度 若不需要改样式即可
loading: boolean
// 是否显示折叠按钮(method: 点击按钮执行的事件, time: 动画执行时间(单位:ms) )
hidden: { method: (isShow: boolean)=>void, time: number }
