<template>
    <div class="ztree-class">
        <ul id="chapter" class="ztree"></ul>
        <Button @click="lookTree">查看当前数的结构</Button>
         <Button @click="addParent">添加父节点</Button>
    </div>
</template>

<script>
export default {
    props: {
        zNodes  : { type: Array,  required: true }, // 章节、知识点
    },
    data() {
        return {
            // zNodes: [
            //     { id:1, pId:0, name:'parent node 1', media:0, open: true },
            //     { id:11, pId:1, name:'leaf node 1-1', mediaDuration:'1200', isMedia:1 },
            //     { id:12, pId:1, name:'leaf node 1-2', mediaDuration:'1500', isMedia:1 },
            //     { id:13, pId:1, name:'leaf node 1-3' },
            //     { id:2, pId:0, name:'parent node 2', open: true },
            //     { id:21, pId:2, name:'leaf node 2-1', mediaDuration:'1900', isMedia:1 },
            //     { id:22, pId:2, name:'leaf node 2-2' },
            //     { id:23, pId:2, name:'leaf node 2-3' },
            //     { id:3, pId:0, name:'parent node 3', open:true },
            //     { id:31, pId:3, name:'leaf node 3-1', mediaDuration:'2100', isMedia:1 },
            //     { id:32, pId:3, name:'leaf node 3-2' },
            //     { id:33, pId:3, name:'leaf node 3-3' },
            //     { id:4, pId:0, name:'parent node 4', open: true },
            // ],
            log: 'dark',
            className: 'dark',
            newCount: 1,
            curDragNodes:{},
            autoExpandNode: {},
            setting:{
                view: {
                    addHoverDom: this.addHoverDom,
                    removeHoverDom: this.removeHoverDom,
                    selectedMulti: false
                },
                edit: {
                    enable: true,
                    editNameSelectAll: true,
                    // showRemoveBtn: false,
                    // showRenameBtn: false,
                },
                data:{
                    simpleData:{
                        enable: true,
                        /*idKey: 'id',
                        pIdKey: 'pId',
                        rootPId: 0*/
                    }
                },
                callback: {
                    beforeDrop: this.beforeDrop,
                    beforeDrag: this.beforeDrag,
                    beforeEditName: this.beforeEditName,
                    beforeRemove: this.beforeRemove,
                    beforeRename: this.beforeRename,
                    onRemove: this.onRemove,
                    onRename: this.onRename,
                    onClick : this.zTreeOnClick,
                    onNodeCreated: this.onNodeCreated,
                    onDrop: this.onDrop,
                }
            },
        };
    },
    computed: {
        chapters() {
            if (this.zNodes.length > 0) {
                for (let node of this.zNodes) {
                    if (node.isMedia == 1) {
                        node.icon = '/static/lib/ztree/css/zTreeStyle/img/diy/3.png';
                    }
                }
            }
            return this.zNodes;
        }
    },
    mounted() {
        $.fn.zTree.init($('#chapter'), this.setting, this.chapters);
        // 查找第一个节点选中
        let treeObj = $.fn.zTree.getZTreeObj('chapter');
        // 根据ID找到该节点
        let node = treeObj.getNodeByParam('id', this.chapters[0].id);
        // 根据该节点选中
        treeObj.selectNode(node);
        this.zTreeOnClick(null, this.chapters[0].id, this.chapters[0]);
    },
    methods: {
        addParent() {
            let node = { id:this.newCount++, pId:0, name:'parent node '+this.newCount, media:0, open: true };
            this.zNodes.push(node);
            // 将父节点入库
            alert('父节点已经入库');
            // 重新初始化树
            $.fn.zTree.init($('#chapter'), this.setting, this.chapters);
        },
        onNodeCreated(event, treeId, genNode) {
            // 如果没有找到节点为新建的节点同步到数据库
            if (this.zNodes.findIndex((x) => x.id == genNode.id) == -1) {
                alert('我已经入库');
            }
        },
        lookTree() {
            console.log(JSON.stringify(this.chapters));
        },
        // 点击节点事件
        zTreeOnClick(event, treeId, treeNode) {
            // 发射事件返回选中数据
            this.$emit('selectedNode', treeNode);
        },
        addHoverDom(treeId, treeNode) {
            // 视频下面不添加节点
            if (treeNode.isMedia == 1) {
                return;
            }
            let self = this;
            let sObj = $('#' + treeNode.tId + '_span');
            if (treeNode.editNameFlag || $('#addBtn_'+treeNode.tId).length>0) return;
            let addStr = "<span class='button add' id='addBtn_" + treeNode.tId
            + "' title='add node' onfocus='this.blur();'></span>";
            sObj.after(addStr);
            let btn = $('#addBtn_'+treeNode.tId);
            if (btn) {
                btn.bind('click', function() {
                    let zTree = $.fn.zTree.getZTreeObj('chapter');
                    zTree.addNodes(treeNode, { id:(100 + self.newCount), pId:treeNode.id, name:'new node' + self.newCount++ });
                    return false;
                });
            }
        },
        removeHoverDom(treeId, treeNode) {
            $('#addBtn_'+treeNode.tId).unbind().remove();
        },
        // 拖拽后触发事件
        beforeDrop(treeId, treeNodes, targetNode, moveType) {
            // 视频不能拖拽为跟节点
            if (targetNode.isMedia == 1) {
                this.$Message.warning('视频不能成为根节点');
                return false;
            }
            return targetNode ? targetNode.drop !== false : true;
        },
        // 拖拽节点操作结束后
        onDrop(event, treeId, treeNodes, targetNode, moveType, isCopy) {
            console.log(treeNodes);
            if (treeNodes.length > 0) {
                // 如果节点拖拽后有父节点变化
                let index = this.zNodes.findIndex((x) => x.id == treeNodes[0].id);
                let thisNode = this.zNodes[index];
                if (treeNodes[0].pId != this.zNodes[index].pId) {
                    this.zNodes[index].pId = treeNodes[0].pId;
                }
                // 获取当前父节点下，除拖拽节点外的所有子节点排序
                let nodes = this.zNodes.filter(x=> x.pId == treeNodes[0].pId);
                // 拖拽后查看上下节点更改节点顺序
                let topNodes = $('#'+treeNodes[0].tId).prevAll();
                let bottomNodes = $('#'+treeNodes[0].tId).nextAll();
                // 如果上面没有兄弟节点则为该父节点的第一个节点
                if (topNodes.length == 0) {
                    thisNode.sort = 0;
                    nodes.push(thisNode);
                    nodes.sort(compare('sort'))
                }
                // 如果下面没有兄弟节点为最末节点
                if (bottomNodes.length == 0) {

                }
                alert(topNodes.length);
                alert(bottomNodes.length);
            }
        },
        compare(property){
            return function(a,b){
                let value1 = a[property];
                let value2 = b[property];
                return value1 - value2;
            }
        },
        // 拖拽前触发事件
        beforeDrag(treeId, treeNodes) {
            let l = treeNodes.length;
            for (let i=0; i < l; i++) {
                if (treeNodes[i].drag === false) {
                    return false;
                }
            }
            return true;
        },
        beforeEditName(treeId, treeNode) {
            this.className = (this.className === 'dark' ? '':'dark');
            this.showLog('[ '+this.getTime()+' beforeEditName ]&nbsp;&nbsp;&nbsp;&nbsp; ' + treeNode.name);
            let zTree = $.fn.zTree.getZTreeObj('chapter');
            zTree.selectNode(treeNode);
            zTree.editName(treeNode);
        },
        // 删除之前调用
        beforeRemove(treeId, treeNode) {
            this.className = (this.className === 'dark' ? '':'dark');
            this.showLog('[ '+this.getTime()+' beforeRemove ]&nbsp;&nbsp;&nbsp;&nbsp; ' + treeNode.name);
            let zTree = $.fn.zTree.getZTreeObj('chapter');
            zTree.selectNode(treeNode);
            return confirm('是否删除节点' + treeNode.name + '?');
        },
        beforeRename(treeId, treeNode, newName, isCancel) {
            this.className = (this.className === 'dark' ? '':'dark');
            this.showLog((isCancel ? '<span style="color:red">':'') + '[ '+this.getTime()+' beforeRename ]&nbsp;&nbsp;&nbsp;&nbsp; ' + treeNode.name + (isCancel ? '</span>':''));
            if (newName.length == 0) {
                setTimeout(function() {
                    let zTree = $.fn.zTree.getZTreeObj('chapter');
                    zTree.cancelEditName();
                    this.$Message.warning('内容不能为空');
                }, 0);
                return false;
            }
            return true;
        },
        // 点击确认按钮删除数据 treeNode 删除掉的数据
        onRemove(e, treeId, treeNode) {
            alert('数据库删除已删除数据');
            // this.showLog('[ '+this.getTime()+' onRemove ]&nbsp;&nbsp;&nbsp;&nbsp; ' + treeNode.name);
        },
        onRename(e, treeId, treeNode, isCancel) {
            this.showLog((isCancel ? '<span style="color:red">':'') + '[ '+this.getTime()+' onRename ]&nbsp;&nbsp;&nbsp;&nbsp; ' + treeNode.name + (isCancel ? '</span>':''));
        },
        showLog(str) {
            if (this.log) {
                this.log = $('#log');
            }
            this.log.append('<li class="this.className">'+str+'</li>');
            if (this.log.children('li').length > 8) {
                this.log.get(0).removeChild(this.log.children('li')[0]);
            }
        },
        getTime() {
            let now = new Date();
            let h = now.getHours();
            let m = now.getMinutes();
            let s = now.getSeconds();
            let ms = now.getMilliseconds();
            return (h + ':' + m + ':' + s + ' ' + ms);
        }
    },
};
</script>
<style lang='scss'>
</style>
