<template>
  <!-- 
    注: 
      1> 若要使用selectedKeys 需要在父组件的@select中进行赋值 当前选中的节点key会作为参数传到select函数中
        const select = (selectedKeysParam: string[]) => {
          selectedKeys.value = selectedKeysParam;
        };
      2> 该组件内含 a-card和a-spin 若有需求 自行在html结构中去掉即可.
      若去掉a-card 隐藏按钮的样式需要微调 
  -->
  <!-- 
    api
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
  -->
  <div class="tree-content" :class="isShow ? '' : 'tree-content-hidden'">
    <a-spin :spinning="loading">
      <a-input
        placeholder="请输入目录名称"
        v-model:value="keyword"
        v-if="props.showSearch"
        style="margin-bottom: 8px;"
      ></a-input>
      <a-tree
        v-if="defaultExpandAll ? treeData && treeData.length > 0 : true"
        :treeData="treeData"
        :replace-fields="replaceFields"
        :expandedKeys="expandedKeys"
        :auto-expand-parent="autoExpandParent"
        @expand="expand"
        @select="select"
        :draggable="dropObj ? true : false"
        @drop="drop"
        v-show="isShow"
        class="tree-operation-class"
        :selectedKeys="selectedKeys"
      >
        <!-- 展示内容 -->
        <template #title="{ dataRef }">
          <span class="title-parent">
            <!-- 检索的情况 -->
            <span
              v-if="
                  props.showSearch &&
                  dataRef[replaceFields.title].indexOf(keyword) > -1
                "
              class="name"
              :title="dataRef[replaceFields.title]"
            >
              {{
              dataRef[replaceFields.title].substr(
              0,
              dataRef[replaceFields.title].indexOf(keyword)
              )
              }}
              <span
                style="color: #f50"
              >{{ keyword }}</span>
              {{
              dataRef[replaceFields.title].substr(
              dataRef[replaceFields.title].indexOf(keyword) +
              keyword.length
              )
              }}
            </span>
            <!-- 默认情况 -->
            <span v-else class="name" :title="dataRef[replaceFields.title]">
              {{
              dataRef[replaceFields.title]
              }}
            </span>
            <!-- 节点操作 -->
            <span
              class="operation-parent"
              v-if="
                  operationList &&
                  operationList.length > 0 &&
                  !noOperationNode.find(
                    (key) => key === dataRef[replaceFields.key]
                  ) &&
                  operationList.find((item) => item.flag(dataRef))
                "
            >
              <a-dropdown
                :getPopupContainer="
                    (triggerNode) =>
                      triggerNode.parentNode.parentNode.parentNode.parentNode
                  "
              >
                <!-- 图标部分 可以改变 -->
                <span @click.stop>
                  <MoreOutlined :rotate="90" />
                </span>
                <!-- 菜单部分 -->
                <template #overlay>
                  <a-menu>
                    <div v-for="operation in operationList" :key="operation.name">
                      <a-menu-item
                        v-if="operation.flag(dataRef)"
                        @click.stop="operation.method(dataRef)"
                      >{{ operation.name }}</a-menu-item>
                    </div>
                  </a-menu>
                </template>
              </a-dropdown>
            </span>
          </span>
        </template>
      </a-tree>
    </a-spin>
    <!-- tree隐藏/显示图标 -->
    <div
      class="ant-tree-hidden-parent"
      v-if="showHidden"
      @click="showButtonClick"
      :title="isShow ? '隐藏' : '展开'"
    >
      <!-- 展开时展示 -->
      <div v-if="isShow" class="ant-tree-hidden-icon">
        <MenuFoldOutlined />
      </div>
      <!-- 收回时展示 -->
      <div v-else class="ant-tree-hidden-icon">
        <MenuUnfoldOutlined />
      </div>
    </div>
  </div>
</template>

<script lang='ts' setup>
import { computed, ref } from "@vue/reactivity";
import { watch, defineProps, defineEmits, nextTick } from "@vue/runtime-core";
import { DropEvent } from "ant-design-vue/lib/tree/Tree";

// 菜单项类型
interface OperationItem {
  name: string;
  method: (record: any) => void;
  flag: (record: any) => boolean;
}

// props&emit
const props = defineProps({
  // 树数据 any[]
  treeData: Object,
  // 树替换字段 {key: string, title: string, children: string}
  replaceFields: Object,
  // 树当前选择节点keys string[]
  selectedKeys: Object,
  // 是否默认展开全部节点
  defaultExpandAll: Boolean,

  // 是否展示搜索框
  showSearch: Boolean,
  // 菜单功能项数组 {name: string, method: (record: 当前节点对象) => void, flag: (record: any) => boolean }
  operationList: Object,
  // 不需要菜单功能的节点key值 string[]
  noOperationNode: Object,
  // 拖拽相关对象 { method: (data: 拖拽完成后的数据, info: DropEvent) => void }
  drop: Object,
  // 是否加载状态
  loading: Boolean,
  // 是否显示折叠按钮 { method: (isShow: boolean)=>void, time: number }
  hidden: Object,
  // 需要展开的节点key数组(只需要传需要展开的节点key即可, 之前展开的不需要再传, 每次都传最新的)
  openExpandedKeys: Array,
});
const emit = defineEmits(["update:selectedKeys", "select"]);

// 数据相关
function data() {
  const treeData = computed(() => props.treeData || []);
  const replaceFields = computed(() => {
    return {
      key: props.replaceFields?.key || "key",
      title: props.replaceFields?.title || "title",
      children: props.replaceFields?.children || "children",
    };
  });
  const defaultExpandAll = computed(() => props.defaultExpandAll);
  const loading = computed(() => props.loading);

  return {
    treeData,
    replaceFields,
    defaultExpandAll,
    loading,
  };
}
const { treeData, replaceFields, defaultExpandAll, loading } = data();

// 选择相关
function treeSelect() {
  const select = (selectedKeys: any[], e: any) => {
    emit("update:selectedKeys", selectedKeys);
    emit("select", e?.node?.dataRef?.publishVersion || "");
  };

  return {
    select,
  };
}
const { select } = treeSelect();

// 树节点展开相关
function treeExpand() {
  // 展开项
  const expandedKeys = ref<string[]>([]);
  // 是否自动展开父节点
  const autoExpandParent = ref(true);
  // @expand
  const expand = (keys: string[]) => {
    expandedKeys.value = keys;
    autoExpandParent.value = false;
  };
  // 默认展开全部(监听loading 当loading为false时执行)
  const loadingWatch = watch(
    () => props.loading,
    (value) => {
      if (!value) {
        nextTick(() => {
          if (defaultExpandAll.value) {
            keyword.value = " ";
            setTimeout(() => {
              keyword.value = "";
            }, 0);
          }
        });
      }
    }
  );
  // 监听props内的openExpandedKeys, 若有改变 添加进expandedKeys
  const openExpandedKeysWatch = watch(
    () => props.openExpandedKeys,
    (value) => {
      if (value && value.length > 0) {
        expandedKeys.value.push(...(value as string[]));
        autoExpandParent.value = false;
      }
    }
  );

  return {
    expandedKeys,
    autoExpandParent,
    expand,
    loadingWatch,
    openExpandedKeysWatch,
  };
}
const {
  expandedKeys,
  autoExpandParent,
  expand,
  loadingWatch,
  openExpandedKeysWatch,
} = treeExpand();

// 拖拽相关
function treeDrag() {
  const dropObj = computed(() => {
    return props.drop;
  });
  // @drop
  const drop = (info: DropEvent) => {
    const dropKey = info.node.eventKey;
    const dragKey = info.dragNode.eventKey;
    const dropPos = info.node.pos.split("-");
    const dropPosition =
      info.dropPosition - Number(dropPos[dropPos.length - 1]);
    const data = [...(treeData.value as any[])];
    // Find dragObject
    let dragObj: any = {};
    loop(data, dragKey, (item: any, index: number, arr: any[]) => {
      arr.splice(index, 1);
      dragObj = item;
    });
    if (!info.dropToGap) {
      // Drop on the content
      loop(data, dropKey, (item: any) => {
        item[replaceFields.value.children] =
          item[replaceFields.value.children] || [];
        // where to insert 示例添加到尾部，可以是随意位置
        item[replaceFields.value.children].push(dragObj);
      });
    } else if (
      (info.node[replaceFields.value.children] || []).length > 0 && // Has children
      info.node.expanded && // Is expanded
      dropPosition === 1 // On the bottom gap
    ) {
      loop(data, dropKey, (item: any) => {
        item[replaceFields.value.children] =
          item[replaceFields.value.children] || [];
        // where to insert 示例添加到尾部，可以是随意位置
        item[replaceFields.value.children].unshift(dragObj);
      });
    } else {
      let ar: any[] = [];
      let i = 0;
      loop(data, dropKey, (item: any, index: number, arr: any[]) => {
        ar = arr;
        i = index;
      });
      if (dropPosition === -1) {
        ar.splice(i, 0, dragObj);
      } else {
        ar.splice(i + 1, 0, dragObj);
      }
    }
    dropObj.value?.method(data, info);
  };

  return {
    dropObj,
    drop,
  };
}
const { dropObj, drop } = treeDrag();

// 检索相关
function treeSearch() {
  // 检索关键字
  const keyword = ref("");
  const treeDataBySearch = computed(() => {
    return splitTreeBySearch(treeData.value as any[], []);
  });
  const keywrodWatch = watch(keyword, (value) => {
    const expanded = treeDataBySearch.value
      .map((item) => {
        if (item[replaceFields.value.title].indexOf(value) > -1) {
          return getParentKey(
            item[replaceFields.value.key],
            treeData.value as any[]
          );
        }
        return undefined;
      })
      .filter(
        (item, i, self) => item !== undefined && self.indexOf(item) === i
      );
    expandedKeys.value = expanded as string[];
    keyword.value = value;
    autoExpandParent.value = true;
  });
  return {
    keyword,
    treeDataBySearch,
    keywrodWatch,
  };
}
const { keyword, treeDataBySearch, keywrodWatch } = treeSearch();

// 树隐藏相关
function treeShowHidden() {
  // 是否显示展开/隐藏图标
  const showHidden = computed(() => (props.hidden ? true : false));
  // 是否显示树
  const isShow = ref(true);
  // 显示/隐藏按钮点击事件
  const showButtonClick = () => {
    const isShowTemp = !isShow.value;
    props.hidden?.method(isShowTemp);
    // 作用于动画
    if (!isShowTemp) {
      setTimeout(() => {
        isShow.value = isShowTemp;
      }, props.hidden?.time || 1000);
    } else isShow.value = isShowTemp;
  };

  return {
    showHidden,
    isShow,
    showButtonClick,
  };
}
const { showHidden, isShow, showButtonClick } = treeShowHidden();

// 操作相关
function treeOperation() {
  // 操作的菜单项 编辑删除新增等
  const operationList = computed((): OperationItem[] => {
    return (props.operationList as OperationItem[]) || [];
  });
  const noOperationNode = computed((): string[] => {
    return (props.noOperationNode as string[]) || [];
  });

  return {
    operationList,
    noOperationNode,
  };
}
const { operationList, noOperationNode } = treeOperation();

function commonMethods() {
  // 将树结构拆分成一维数组(标准节点对象)
  const splitTreeBySearch = (treeNode: any[], treeNodeList: any[]) => {
    treeNode.forEach((item) => {
      const children = item[replaceFields.value.children];
      treeNodeList.push(item);
      if (children) {
        splitTreeBySearch(children, treeNodeList);
      }
    });
    return treeNodeList;
  };

  // 获取当前节点的父节点key(用于用户传来的节点对象)
  const getParentKey = (
    key: string,
    tree: any[]
  ): string | number | undefined => {
    let parentKey;
    for (let i = 0; i < tree.length; i++) {
      const node = tree[i];
      const children = node[replaceFields.value.children];
      if (children) {
        if (
          children.some((item: any) => item[replaceFields.value.key] === key)
        ) {
          parentKey = node[replaceFields.value.key];
        } else if (getParentKey(key, children)) {
          parentKey = getParentKey(key, children);
        }
      }
    }
    return parentKey;
  };

  const loop = (data: any[], key: string, callback: any) => {
    data.forEach((item, index, arr) => {
      if (item[replaceFields.value.key] === key) {
        return callback(item, index, arr);
      }
      if (item[replaceFields.value.children]) {
        return loop(item[replaceFields.value.children], key, callback);
      }
    });
  };

  return {
    splitTreeBySearch,
    getParentKey,
    loop,
  };
}
const { splitTreeBySearch, getParentKey, loop } = commonMethods();
</script>

<style lang='less'>
.tree-content {
  position: relative;
  // 隐藏的样式
  &.tree-content-hidden {
    width: 0px;
    .ant-card-bordered {
      border: 0px;
    }
  }
  // 树加载样式
  .ant-spin.ant-spin-spinning {
    height: 240px;
    line-height: 240px;
    width: 100%;
  }
  // 添加根节点图标
  .not-add-root-icon {
    float: right;
    line-height: 25px;
    cursor: pointer;
    &:hover {
      color: #066bd7;
    }
  }
  // 操作节点样式
  .tree-operation-class {
    // node
    .ant-tree-node-content-wrapper {
      width: calc(100% - 24px) !important;
      display: inline-block !important;
      position: relative;
      line-height: 28px;
      height: 32px;
      // hover时显示操作选项
      &:hover .ant-tree-title .title-parent .operation-parent {
        opacity: 1;
      }
      // 树节点文本
      .ant-tree-title {
        width: 100% !important;
        display: inline-block !important;
        .title-parent {
          display: flex;
          width: 100%;
          position: relative;
          // title样式
          .name {
            white-space: nowrap;
            text-overflow: ellipsis;
            overflow: hidden;
            flex: 1;
          }
          // 操作项样式
          .operation-parent {
            width: 20px;
            float: right;
            opacity: 0;
            color: rgba(0, 0, 0, 70%);
            .operation-icon {
              &:hover {
                color: #066bd7;
              }
            }
          }
        }
      }
      // 调整选中颜色
      &.ant-tree-node-selected {
        background-color: #066bd71a !important;
        color: rgb(6, 107, 215);
      }
    }
    // li
    li {
      // 调整节点之间的间距
      padding: 2px 0;
      .ant-tree-child-tree > li:first-child {
        padding-top: 4px;
      }
      // 调整节点左侧icon的高度
      span.ant-tree-switcher,
      span.ant-tree-switcher {
        height: 32px;
        line-height: 28px;
      }
    }
    // 下拉菜单
    .ant-dropdown-menu {
      margin: 0px !important;
      padding: 4px 0 !important;
      list-style-type: none !important;
      .ant-dropdown-menu-item {
        margin: 0 !important;
        padding: 5px 12px !important;
        white-space: nowrap !important;
        &::before {
          content: "";
          border-left: 0px !important;
        }
      }
    }
  }
  // 隐藏/显示图标样式
  .ant-tree-hidden-parent {
    width: 24px;
    height: 100%;
    opacity: 1;
    text-align: center;
    position: absolute;
    top: 0;
    right: -12px;
    .ant-tree-hidden-icon {
      width: 24px;
      height: 24px;
      background: #fff;
      border-radius: 50%;
      box-shadow: 0px 1px 4px 0px rgba(38, 38, 38, 0.1);
      color: #066bd7;
      font-size: 17px;
      cursor: pointer;
      position: absolute;
      top: 50%;
      transform: translateY(-50%);
      &:hover {
        background-color: #066bd7;
        color: #fff;
      }
    }
  }
}
</style>