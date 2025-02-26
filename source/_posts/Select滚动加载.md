---
title: Select滚动加载
tags:
  - Vue
  - 前端
date: 2025-02-11 21:29:25
---


## 需求

项目中有些表单中关联的数据项的数据较多（几千条），如果全部查询出来太占内存，造成页面卡顿，需要进行分页查询。

解决方案：

1. 弹窗中嵌入表格供用户查询选择
2. Select滚动加载

本篇文章介绍的是 Select 滚动加载。思路：获取选项（Option）区域 DOM 元素 --> 添加滚动(scroll)事件监听 --> 比较总高度、可视区域高度、剩余高度，注意在组件销毁时要移除事件绑定，避免内存泄漏。

## 原生API

打开 select 下拉列表绑定滚动（scroll）事件，关闭下拉列表时移除事件绑定。

这里需要注意几个点：

1. 如果初始元素过少无法滚动，也就无法触发滚动事件
2. DOM 元素需要选对，Option 的选项框
3. target.scrollHeight - target.scrollTop - target.clientHeight <= 1 中需要与 1 进行比较，不然可能出现小数大于0不满足条件的情况

```html
<template>
  <el-select
    v-model="value"
    placeholder="Select"
    size="large"
    style="width: 240px"
    @visible-change="handleVisibleChange"
  >
    <el-option
      v-for="item in options"
      :label="item.label"
      :value="item.value"
    />
  </el-select>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      value: null,
      options: [],
    };
  },
  methods: {
    handleVisibleChange(visible) {
      if (visible) {
        const dom = document.querySelector(
            ".el-select-dropdown .el-select-dropdown__wrap"
          );
          dom.addEventListener("scroll", this.handleScroll);
      } else {
        const dom = document.querySelector(
          ".el-select-dropdown .el-select-dropdown__wrap"
        );
        dom.removeEventListener("scroll", this.handleScroll);
      }
    },
    handleScroll(event) {
      const target = event.target;
      /**
       * scrollHeight 获取元素内容高度(只读)
       * scrollTop 获取或者设置元素的偏移值,
       *  常用于:计算滚动条的位置, 当一个元素的容器没有产生垂直方向的滚动条, 那它的scrollTop的值默认为0.
       * clientHeight 读取元素的可见高度(只读)
       * 如果元素滚动到底, 下面等式返回true, 没有则返回false:
       * ele.scrollHeight - ele.scrollTop === ele.clientHeight;
       */
      const condition =
        target.scrollHeight - target.scrollTop - target.clientHeight <= 1;
      if (condition) {
        for (let i = 0; i < 10; i++) {
          const item = "Option" + i;
          this.options.push({ label: item, value: item });
        }
      }
    },
  },
  created() {
    for (let i = 0; i < 20; i++) {
      const item = "Option" + i;
      this.options.push({ label: item, value: item });
    }
  },
};
</script>
```


## 自定义指令

注意事项：不是通过 el 获取 DOM 元素，因为 option 的选项框不在选择框里面，无法获取到盒子，需要通过 document 获取

```html
<template>
  <el-select
    v-model="value"
    placeholder="Select"
    size="large"
    style="width: 240px"
    v-select-more="loadMore"
  >
    <el-option
      v-for="item in options"
      :label="item.label"
      :value="item.value"
    />
  </el-select>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      value: null,
      options: [],
    };
  },

  directives: {
    "select-more": {
      mounted(el, binding) {
        el._onScroll = function (event) {
          const SELECTWRAP_DOM = event.target;
          if (SELECTWRAP_DOM) {
            const condition =
              SELECTWRAP_DOM.scrollHeight -
                SELECTWRAP_DOM.scrollTop -
                SELECTWRAP_DOM.clientHeight <=
              1;
            if (condition && typeof binding.value === "function") {
              binding.value();
            }
          }
        };

        // 获取元素
        const SELECTWRAP_DOM = document.querySelector(
          ".el-select-dropdown .el-select-dropdown__wrap"
        );

        if (SELECTWRAP_DOM) {
          SELECTWRAP_DOM.addEventListener("scroll", el._onScroll);
        }
      },
      unmounted(el) {
        // 在组件销毁时移除滚动监听器
        const SELECTWRAP_DOM = el.querySelector(
          ".el-select-dropdown .el-select-dropdown__wrap"
        );
        // 清理引用以避免内存泄漏
        if (SELECTWRAP_DOM && el._onScroll) {
          SELECTWRAP_DOM.removeEventListener("scroll", el._onScroll);
          delete el._onScroll;
        }
      },
    },
  },
  methods: {
    loadMore() {
      for (let i = 0; i < 20; i++) {
        const item = "Option" + i;
        this.options.push({ label: item, value: item });
      }
    },
  },
  created() {
    for (let i = 0; i < 20; i++) {
      const item = "Option" + i;
      this.options.push({ label: item, value: item });
    }
  },
};
</script>
```

## 参考文章

https://blog.csdn.net/weixin_46022934/article/details/138976308