<template>
  <el-table
    ref="table"
    :data="tableViewData"
    style="width: 100%"
    :height="tableHeight"
    :max-height="tableHeight"
    class="custom-table"
    :row-style="{'height':`${rowHeight}px`}"
    :header-row-style="{'height':`${headerHeight}px`}"
    >

    <el-table-column
      type="index"
    />

    <el-table-column
      prop="name"
      label="名字"
    >
      <template slot-scope="scope">
        <!-- 计算缩进，如果有子节点代表还会多出一个展开图标，所以给无子节点的再加上展开图标的大小。 -->
        <div :style="{'margin-left': `${(scope.row._levelNum * 8 + (scope.row._childrenNum>0?0:14))}px`}">

        <!-- 自行实现树形表格的展开功能 -->
          <i 
            v-if="scope.row._childrenNum>0" 
            :class="`${scope.row._isExpand?'el-icon-arrow-down':'el-icon-arrow-right'}`"
            style="cursor: pointer"
            @click="toggleExpand(scope.row)"
          ></i>
          <span >{{ scope.row.name }}</span>
        </div>

      </template>
    </el-table-column>
    <el-table-column
      prop="_levelNum"
      label="层数">
    </el-table-column>
    <el-table-column
      prop="_index"
      label="实际索引">
    </el-table-column>
  </el-table>
</template>

<script>

function traverseTree (data, callback, childrenKey = 'children') {
  // 树形结构遍历
  if (!data || !callback) {
    return console.error('please check parameter:', data, callback)
  }
  data.forEach(item => {
    callback(item)
    const children = item[childrenKey]
    if (children) {
      traverseTree(children, callback, childrenKey)
    }
  })
}

  export default {
    data() {
      const treeData=[
        {
          name:'1',
        },
        {
          name:'2',
        },
        {
          name:'3',
          children:[
            {
              name:'3-1',
              children:[
                {name:'3-1-1'},
                {name:'3-1-2'},
                {name:'3-1-3'},
                {name:'3-1-4'},
              ]
            },
            {
              name:'3-2',
              children:[
                {name:'3-2-1'},
                {name:'3-2-2'},
                {name:'3-2-3'},
                {name:'3-2-4'},
              ]
            }
          ]
        },
        {
          name:'4',
          children:[
            {
              name:'4-1',
              children:[
                {name:'4-1-1'},
                {name:'4-1-2'},
                {name:'4-1-3'},
                {name:'4-1-4'},
              ]
            },
            {
              name:'4-2',
              children:[
                {name:'4-2-1'},
                {name:'4-2-2'},
                {name:'4-2-3'},
                {name:'4-2-4'},
              ]
            }
          ]
        },
        {
          name:'5',
        },
        {
          name:'6',
        },
      ]
      return {
        rowHeight:50, // 表格每行的高度，用于计算定位
        headerHeight:50,// 表格表头高度
        tableHeight:400,// 表格最大高度
        treeData:treeData,// 原始的树形结构对象
        tableData: [],// 转换成平行结构后的数据对象
        tableViewData:[],// 显示到界面的数据
        parentNameObj:{},// name:parentName，用于通过名称找到父级名称
        treeDataObj:{}, // name:{...}，用于通过名称来找回原来的数据对象
        tableDataObj:{},// name:{...}，用于通过名称找到转换后的数据对象
      }
    },
    mounted(){
      this.parseTreeData()
      this.initVirtualElement()
      
      this.updateViewData()
      // 添加滚动事件，滚动时再触发一次计算
      this.$refs.table.bodyWrapper.addEventListener('scroll', this.updateViewData)

      console.log('this.tableData',this.tableData)
    },
    methods: {
      parseTreeData(){
        const tableData = []
        let index = 1
        traverseTree(this.treeData,(i)=>{
          // 转换成平行结构的对象

          const obj = {
            name:i.name,
            _index:index,// 索引数，方便查看调试定位 
            _levelNum:this.getLevelNum(i),// 层数，用于计算缩进
            _isExpand:true,// 默认展开
            _childrenNum:(i.children && i.children.length) || 0
          }
          tableData.push(obj)
          this.tableDataObj[i.name] = obj

          this.treeDataObj[i.name] = i

          index+=1

          // 记录该节点的父级名称
          if(i.children && i.children.length>0){
            i.children.forEach(j=>{
              this.parentNameObj[j.name] = i.name
            })
          }

        })
        this.tableData = tableData
      },
      updateViewData(){
        // 根据是否展开来过滤掉不用显示的元素
        const tableViewData = this.tableData.filter(i=>{
          return this.isShowByParentExpand(i)
        })
        // 计算是显示从第几行开始展示与结束
        const scrollTop = this.$refs.table.bodyWrapper.scrollTop
        let rowStartNum = Math.ceil(scrollTop / this.rowHeight)
        if (!this.showRowNum) {
          this.showRowNum = Math.ceil((this.tableHeight - this.headerHeight) / this.rowHeight)
        }
        // 最大的起始行
        if(rowStartNum>tableViewData.length-this.showRowNum){
          rowStartNum = tableViewData.length-this.showRowNum
        }
        // 最大的结束行
        let rowEndNum = rowStartNum + this.showRowNum
        if(rowEndNum>tableViewData.length){
          rowEndNum = tableViewData.length
        }
        
        console.log('updateTableView row:', rowStartNum, '-', rowEndNum, ',', this.showRowNum,tableViewData.length)
        // 计算上下占位元素的高度
        // const top = rowStartNum * this.rowHeight + 'px'
        // const bottom = (tableViewData.length + 1 - rowEndNum)  * this.rowHeight + 'px'

        let top = Math.round(scrollTop) + 'px'
        const bottom = (tableViewData.length +1 - rowEndNum) * this.rowHeight + 'px'
        // 如果到尾了，上占位达到最大高度
        if(rowStartNum ===  tableViewData.length-this.showRowNum){
          top = (tableViewData.length-this.showRowNum) * this.rowHeight + 'px'
        }
        console.log('top',top,bottom)
        // 调整占位元素的高度
        this.topElement.style.height = top
        this.bottomElement.style.height = bottom
        // 表格只显示要显示的行数内容
        this.tableViewData = tableViewData.slice(rowStartNum,rowEndNum)

      },
      getLevelNum(obj){
        // 通过父级名称来计算该节点是第几层
        let result = 0
        let parentName = obj.name
        do {
          result += 1
          parentName = this.parentNameObj[parentName]
        } while (parentName)
        return result
      },
      isShowByParentExpand(obj){
        // 根据父级是否展开来判断 是否显示
        let flag = true
        let parentName = this.parentNameObj[obj.name]
        while (flag && parentName) {
          const parent = this.tableDataObj[parentName]
          flag = parent._isExpand
          parentName = this.parentNameObj[parent.name]
        }
        return flag
      },
      toggleExpand(row){
        this.$set(row,'_isExpand',!row._isExpand)
        this.updateViewData()
      },
      initVirtualElement(){
        // 表格内部上下放两个占位元素，用于填充空间视图外的空间
        const tbody = this.$refs.table.bodyWrapper.firstChild
        
        this.topElement = document.createElement('div')
        this.topElement.id = 'topElement'
        this.bottomElement = document.createElement('div')
        this.bottomElement.id = 'bottomElement'
        // 插入到最前面
        tbody.insertBefore(this.topElement, tbody.firstChild)
        // 插入到最后面
        tbody.appendChild(this.bottomElement)
      }
    }
  }
</script>

<style>
/* 表格行高 可通过row-style配置*/
/* .custom-table > .el-table__body-wrapper tbody .el-table__row{
  height: 50px;
} */

/* 表头行高 可通过header-row-style配置*/
/* .custom-table > .el-table__header-wrapper .el-table__header thead tr{
  height: 50px;
} */
</style>