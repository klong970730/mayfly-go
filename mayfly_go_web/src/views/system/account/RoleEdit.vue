<template>
    <div class="account-dialog">
        <el-dialog :title="account == null ? '' : '分配“' + account.username + '”的角色'" v-model="dialogVisible" :before-close="cancel" :show-close="false">
            <div class="card pd5">
                <div>
                    <el-input placeholder="请输入角色名" style="width: 150px" v-model="query.name" @clear="clear()" clearable> </el-input>
                    <el-button class="ml5" @click="search" type="success" icon="search"></el-button>
                </div>
            </div>
            <el-table :data="allRole" border ref="roleTable" @select="select" style="width: 100%">
                <el-table-column :selectable="selectable" type="selection" width="40"></el-table-column>
                <el-table-column prop="name" label="角色名称"></el-table-column>
                <el-table-column prop="code" label="角色code"></el-table-column>
                <el-table-column prop="remark" label="角色描述">
                    <template #default="scope">
                        {{ scope.row.remark ? scope.row.remark : '暂无描述' }}
                    </template>
                </el-table-column>
            </el-table>
            <el-row type="flex" justify="end">
                <el-pagination
                    @current-change="handlePageChange"
                    style="text-align: center; margin-top: 20px"
                    background
                    layout="prev, pager, next, total, jumper"
                    :total="total"
                    v-model:current-page="query.pageNum"
                    :page-size="query.pageSize"
                ></el-pagination>
            </el-row>

            <template #footer>
                <div class="dialog-footer">
                    <el-button @click="cancel()">取 消</el-button>
                    <el-button type="primary" :loading="btnLoading" @click="btnOk">确 定</el-button>
                </div>
            </template>
        </el-dialog>
    </div>
</template>

<script lang="ts" setup>
import { toRefs, reactive, watch, ref } from 'vue';
import { roleApi, accountApi } from '../api';
import { ElMessage } from 'element-plus';

const props = defineProps({
    visible: {
        type: Boolean,
    },
    account: Object,
});

//定义事件
const emit = defineEmits(['update:visible', 'cancel', 'val-change']);

const roleTable: any = ref(null);

const state = reactive({
    dialogVisible: false,
    btnLoading: false,
    // 所有角色
    allRole: [] as any,
    // 该账号拥有的角色id
    query: {
        name: null,
        pageNum: 1,
        pageSize: 5,
    },
    total: 0,
});

const { dialogVisible, btnLoading, allRole, query, total } = toRefs(state);

// 用户拥有的角色信息
let roles: any[] = [];

watch(props, (newValue: any) => {
    state.dialogVisible = newValue.visible;
    if (state.dialogVisible && newValue.account && newValue.account.id != 0) {
        accountApi.roleIds
            .request({
                id: props.account!.id,
            })
            .then((res) => {
                roles = res || [];
                search();
            });
    }
});

const handlePageChange = () => {
    search();
};

const selectable = (row: any) => {
    // 角色code不以COMMON开头才可勾选
    return row.code.indexOf('COMMON') != 0;
};

const select = (val: any, row: any) => {
    // 如果账号的角色id存在则为取消该角色(删除角色id列表中的该记录id)，否则为新增角色
    if (roles.includes(row.id)) {
        for (let i = 0; i < roles.length; i++) {
            let item = roles[i];
            if (item === row.id) {
                roles.splice(i, 1);
                break;
            }
        }
    } else {
        roles.push(row.id);
    }
};

/**
 * 检查是否勾选权限,即是否拥有权限
 */
const checkSelected = () => {
    // 必须用异步，否则勾选不了
    setTimeout(() => {
        roleTable.value.clearSelection();
        state.allRole.forEach((r: any) => {
            if (roles.includes(r.id)) {
                roleTable.value.toggleRowSelection(r, true);
            }
        });
    }, 50);
};

const btnOk = async () => {
    let roleIds = roles.join(',');
    await accountApi.saveRoles.request({
        id: props.account!.id,
        roleIds: roleIds,
    });
    ElMessage.success('保存成功!');
    cancel();
};

/**
 * 取消
 */
const cancel = () => {
    state.query.pageNum = 1;
    state.query.name = null;
    emit('update:visible', false);
    emit('cancel');
};

/**
 * 清空查询框
 */
const clear = () => {
    state.query.pageNum = 1;
    state.query.name = null;
    search();
};

const search = async () => {
    let res = await roleApi.list.request(state.query);
    state.allRole = res.list;
    state.total = res.total;
    checkSelected();
};
</script>
