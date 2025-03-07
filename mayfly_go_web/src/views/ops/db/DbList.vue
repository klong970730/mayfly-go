<template>
    <div class="db-list">
        <page-table
            ref="pageTableRef"
            :page-api="dbApi.dbs"
            :before-query-fn="checkRouteTagPath"
            :search-items="searchItems"
            v-model:query-form="query"
            :show-selection="true"
            v-model:selection-data="state.selectionData"
            :columns="columns"
        >
            <template #tagPathSelect>
                <el-select @focus="getTags" v-model="query.tagPath" placeholder="请选择标签" filterable clearable>
                    <el-option v-for="item in tags" :key="item" :label="item" :value="item"> </el-option>
                </el-select>
            </template>

            <template #instanceSelect>
                <el-select remote :remote-method="getInstances" v-model="query.instanceId" placeholder="输入并选择实例" filterable clearable>
                    <el-option v-for="item in state.instances" :key="item.id" :label="`${item.name}`" :value="item.id">
                        {{ item.name }}
                        <el-divider direction="vertical" border-style="dashed" />

                        {{ item.type }} / {{ item.host }}:{{ item.port }}
                        <el-divider direction="vertical" border-style="dashed" />
                        {{ item.username }}
                    </el-option>
                </el-select>
            </template>

            <template #tableHeader>
                <el-button v-auth="perms.saveDb" type="primary" icon="plus" @click="editDb(false)">添加</el-button>
                <el-button v-auth="perms.delDb" :disabled="selectionData.length < 1" @click="deleteDb()" type="danger" icon="delete">删除</el-button>
            </template>

            <template #type="{ data }">
                <el-tooltip :content="data.type" placement="top">
                    <SvgIcon :name="getDbDialect(data.type).getInfo().icon" :size="20" />
                </el-tooltip>
            </template>

            <template #host="{ data }">
                {{ `${data.host}:${data.port}` }}
            </template>

            <template #tagPath="{ data }">
                <resource-tag :resource-code="data.code" :resource-type="TagResourceTypeEnum.Db.value" />
            </template>

            <template #action="{ data }">
                <span v-if="actionBtns[perms.saveDb]">
                    <el-button type="primary" @click="editDb(data)" link>编辑</el-button>
                    <el-divider direction="vertical" border-style="dashed" />
                </span>

                <el-button type="primary" @click="onShowSqlExec(data)" link>SQL记录</el-button>
                <el-divider direction="vertical" border-style="dashed" />

                <el-dropdown @command="handleMoreActionCommand">
                    <span class="el-dropdown-link-more">
                        更多
                        <el-icon class="el-icon--right">
                            <arrow-down />
                        </el-icon>
                    </span>
                    <template #dropdown>
                        <el-dropdown-menu>
                            <el-dropdown-item :command="{ type: 'detail', data }"> 详情 </el-dropdown-item>

                            <!-- <el-dropdown-item :command="{ type: 'edit', data }" v-if="actionBtns[perms.saveDb]"> 编辑 </el-dropdown-item> -->

                            <el-dropdown-item :command="{ type: 'dumpDb', data }" v-if="data.type == DbType.mysql"> 导出 </el-dropdown-item>
                        </el-dropdown-menu>
                    </template>
                </el-dropdown>
            </template>
        </page-table>

        <el-dialog width="720px" :title="`${db} 数据库导出`" v-model="exportDialog.visible">
            <el-row justify="space-between">
                <el-col :span="9">
                    <el-form-item label="导出内容: ">
                        <el-checkbox-group v-model="exportDialog.contents" :min="1">
                            <el-checkbox label="结构" />
                            <el-checkbox label="数据" />
                        </el-checkbox-group>
                    </el-form-item>
                </el-col>
                <el-col :span="9">
                    <el-form-item label="扩展名: ">
                        <el-radio-group v-model="exportDialog.extName">
                            <el-radio label="sql" />
                            <el-radio label="gzip" />
                        </el-radio-group>
                    </el-form-item>
                </el-col>
            </el-row>

            <el-form-item>
                <el-transfer
                    v-model="exportDialog.value"
                    filterable
                    filter-placeholder="按数据库名称筛选"
                    :titles="['全部数据库', '导出数据库']"
                    :data="exportDialog.data"
                    max-height="300"
                    size="small"
                />
            </el-form-item>

            <template #footer>
                <div class="dialog-footer">
                    <el-button @click="exportDialog.visible = false">取消</el-button>
                    <el-button @click="dumpDbs()" type="primary">确定</el-button>
                </div>
            </template>
        </el-dialog>

        <el-dialog
            width="90%"
            :title="`${sqlExecLogDialog.title} - SQL执行记录`"
            :before-close="onBeforeCloseSqlExecDialog"
            :close-on-click-modal="false"
            v-model="sqlExecLogDialog.visible"
            :destroy-on-close="true"
        >
            <db-sql-exec-log :db-id="sqlExecLogDialog.dbId" :dbs="sqlExecLogDialog.dbs" />
        </el-dialog>

        <el-dialog v-model="infoDialog.visible" :before-close="onBeforeCloseInfoDialog" :close-on-click-modal="false">
            <el-descriptions title="详情" :column="3" border>
                <!-- <el-descriptions-item :span="3" label="标签路径">{{ infoDialog.data?.tagPath }}</el-descriptions-item> -->
                <el-descriptions-item :span="2" label="名称">{{ infoDialog.data?.name }}</el-descriptions-item>
                <el-descriptions-item :span="1" label="id">{{ infoDialog.data?.id }}</el-descriptions-item>
                <el-descriptions-item :span="3" label="数据库">{{ infoDialog.data?.database }}</el-descriptions-item>
                <el-descriptions-item :span="3" label="备注">{{ infoDialog.data?.remark }}</el-descriptions-item>
                <el-descriptions-item :span="2" label="创建时间">{{ dateFormat(infoDialog.data?.createTime) }} </el-descriptions-item>
                <el-descriptions-item :span="1" label="创建者">{{ infoDialog.data?.creator }}</el-descriptions-item>
                <el-descriptions-item :span="2" label="更新时间">{{ dateFormat(infoDialog.data?.updateTime) }} </el-descriptions-item>
                <el-descriptions-item :span="1" label="修改者">{{ infoDialog.data?.modifier }}</el-descriptions-item>

                <el-descriptions-item :span="3" label="数据库实例名称">{{ infoDialog.instance?.name }}</el-descriptions-item>
                <el-descriptions-item :span="2" label="主机">{{ infoDialog.instance?.host }}</el-descriptions-item>
                <el-descriptions-item :span="1" label="端口">{{ infoDialog.instance?.port }}</el-descriptions-item>
                <el-descriptions-item :span="2" label="用户名">{{ infoDialog.instance?.username }}</el-descriptions-item>
                <el-descriptions-item :span="1" label="类型">{{ infoDialog.instance?.type }}</el-descriptions-item>
            </el-descriptions>
        </el-dialog>

        <db-edit @val-change="search" :title="dbEditDialog.title" v-model:visible="dbEditDialog.visible" v-model:db="dbEditDialog.data"></db-edit>
    </div>
</template>

<script lang="ts" setup>
import { ref, toRefs, reactive, onMounted, defineAsyncComponent, Ref } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import { dbApi } from './api';
import config from '@/common/config';
import { joinClientParams } from '@/common/request';
import { isTrue } from '@/common/assert';
import { dateFormat } from '@/common/utils/date';
import ResourceTag from '../component/ResourceTag.vue';
import PageTable from '@/components/pagetable/PageTable.vue';
import { TableColumn } from '@/components/pagetable';
import { hasPerms } from '@/components/auth/auth';
import DbSqlExecLog from './DbSqlExecLog.vue';
import { DbType } from './dialect';
import { tagApi } from '../tag/api';
import { TagResourceTypeEnum } from '@/common/commonEnum';
import { useRoute } from 'vue-router';
import { getDbDialect } from './dialect/index';
import { SearchItem } from '@/components/SearchForm';

const DbEdit = defineAsyncComponent(() => import('./DbEdit.vue'));

const perms = {
    base: 'db',
    saveDb: 'db:save',
    delDb: 'db:del',
};

const searchItems = [SearchItem.slot('tagPath', '标签', 'tagPathSelect'), SearchItem.slot('instanceId', '实例', 'instanceSelect')];

const columns = ref([
    TableColumn.new('instanceName', '实例名'),
    TableColumn.new('type', '类型').isSlot().setAddWidth(-15).alignCenter(),
    TableColumn.new('host', 'ip:port').isSlot().setAddWidth(40),
    TableColumn.new('username', 'username'),
    TableColumn.new('name', '名称'),
    TableColumn.new('tagPath', '关联标签').isSlot().setAddWidth(10).alignCenter(),
    TableColumn.new('remark', '备注'),
]);

// 该用户拥有的的操作列按钮权限
const actionBtns = hasPerms([perms.base, perms.saveDb]);
const actionColumn = TableColumn.new('action', '操作').isSlot().setMinWidth(220).fixedRight().alignCenter();

const route = useRoute();
const pageTableRef: Ref<any> = ref(null);
const state = reactive({
    row: {} as any,
    dbId: 0,
    db: '',
    tags: [],
    instances: [] as any,
    /**
     * 选中的数据
     */
    selectionData: [],
    /**
     * 查询条件
     */
    query: {
        tagPath: '',
        instanceId: null,
        pageNum: 1,
        pageSize: 0,
    },
    infoDialog: {
        visible: false,
        data: null as any,
        instance: null as any,
        query: {
            instanceId: 0,
        },
    },
    // sql执行记录弹框
    sqlExecLogDialog: {
        title: '',
        visible: false,
        dbs: [],
        dbId: 0,
    },
    exportDialog: {
        visible: false,
        dbId: 0,
        type: 3,
        data: [] as any,
        value: [],
        contents: [] as any,
        extName: '',
    },
    dbEditDialog: {
        visible: false,
        data: null as any,
        title: '新增数据库',
    },
    filterDb: {
        param: '',
        cache: [],
        list: [],
    },
});

const { db, tags, selectionData, query, infoDialog, sqlExecLogDialog, exportDialog, dbEditDialog } = toRefs(state);

onMounted(async () => {
    if (Object.keys(actionBtns).length > 0) {
        columns.value.push(actionColumn);
    }
});

const checkRouteTagPath = (query: any) => {
    if (route.query.tagPath) {
        query.tagPath = route.query.tagPath as string;
    }
    return query;
};

const search = async () => {
    pageTableRef.value.search();
};

const showInfo = async (info: any) => {
    state.infoDialog.data = info;
    state.infoDialog.query.instanceId = info.instanceId;
    const res = await dbApi.getInstance.request(state.infoDialog.query);
    state.infoDialog.instance = res;
    state.infoDialog.visible = true;
};

const onBeforeCloseInfoDialog = () => {
    state.infoDialog.visible = false;
    state.infoDialog.data = null;
    state.infoDialog.instance = null;
};

const getTags = async () => {
    state.tags = await tagApi.getResourceTagPaths.request({ resourceType: TagResourceTypeEnum.Db.value });
};

const getInstances = async (instanceName = '') => {
    if (!instanceName) {
        state.instances = [];
        return;
    }
    const data = await dbApi.instances.request({ name: instanceName });
    if (data) {
        state.instances = data.list;
    }
};

const handleMoreActionCommand = (commond: any) => {
    const data = commond.data;
    const type = commond.type;
    switch (type) {
        case 'detail': {
            showInfo(data);
            return;
        }
        case 'edit': {
            editDb(data);
            return;
        }
        case 'dumpDb': {
            onDumpDbs(data);
        }
    }
};

const editDb = async (data: any) => {
    if (!data) {
        state.dbEditDialog.data = null;
        state.dbEditDialog.title = '新增数据库资源';
    } else {
        state.dbEditDialog.data = data;
        state.dbEditDialog.title = '修改数据库资源';
    }
    state.dbEditDialog.visible = true;
};

const deleteDb = async () => {
    try {
        await ElMessageBox.confirm(`确定删除【${state.selectionData.map((x: any) => x.name).join(', ')}】库?`, '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning',
        });
        await dbApi.deleteDb.request({ id: state.selectionData.map((x: any) => x.id).join(',') });
        ElMessage.success('删除成功');
        search();
    } catch (err) {
        //
    }
};

const onShowSqlExec = async (row: any) => {
    state.sqlExecLogDialog.title = `${row.name}`;
    state.sqlExecLogDialog.dbId = row.id;
    state.sqlExecLogDialog.dbs = row.database.split(' ');
    state.sqlExecLogDialog.visible = true;
};

const onBeforeCloseSqlExecDialog = () => {
    state.sqlExecLogDialog.visible = false;
    state.sqlExecLogDialog.dbs = [];
    state.sqlExecLogDialog.dbId = 0;
};

const onDumpDbs = async (row: any) => {
    const dbs = row.database.split(' ');
    const data = [];
    for (let name of dbs) {
        data.push({
            key: name,
            label: name,
        });
    }
    state.exportDialog.value = [];
    state.exportDialog.data = data;
    state.exportDialog.dbId = row.id;
    state.exportDialog.contents = ['结构', '数据'];
    state.exportDialog.extName = 'sql';
    state.exportDialog.visible = true;
};

/**
 * 数据库信息导出
 */
const dumpDbs = () => {
    isTrue(state.exportDialog.value.length > 0, '请添加要导出的数据库');
    const a = document.createElement('a');
    let type = 0;
    for (let c of state.exportDialog.contents) {
        if (c == '结构') {
            type += 1;
        } else if (c == '数据') {
            type += 2;
        }
    }
    a.setAttribute(
        'href',
        `${config.baseApiUrl}/dbs/${state.exportDialog.dbId}/dump?db=${state.exportDialog.value.join(',')}&type=${type}&extName=${
            state.exportDialog.extName
        }&${joinClientParams()}`
    );
    a.click();
    state.exportDialog.visible = false;
};
</script>
<style lang="scss">
.db-list {
    .el-transfer-panel {
        width: 250px;
    }
}
.el-dropdown-link-more {
    cursor: pointer;
    color: var(--el-color-primary);
    display: flex;
    align-items: center;
    margin-top: 6px;
}
</style>
