<template>
  <div class="dept-staff-panel">
    <el-row :gutter="16">
      <el-col :span="6">
        <div class="side-card">
          <div class="side-card__header">
            <div class="header-left">可选部门列表</div>
            <div v-if="!deptLoading && deptList.length" class="header-right">
              <el-input v-model="deptSearchKeyword" class="header-search-input" size="mini" clearable placeholder="搜索部门"
                @click.native.stop />
            </div>
          </div>
          <div class="side-card__body">
            <div v-if="deptLoading" class="side-card__placeholder">加载中...</div>
            <div v-else-if="!deptList.length" class="side-card__placeholder">暂无部门</div>
            <template v-else>
              <div v-if="!filteredDeptList.length" class="side-card__placeholder">无匹配部门</div>
              <div v-else class="dept-checkbox-group">
                <div v-for="(dept, index) in filteredDeptList" :key="dept.__deptKey || index"
                  :class="['dept-item', { 'dept-item--active': dept.deptId === activeDeptId }]"
                  @click="handleSelectDept(dept)">
                  <div class="dept-item__row">
                    <div class="dept-item__checkbox"
                      :class="{
                        'dept-item__checkbox--checked': getDeptActiveStatus(dept.deptId),
                        'dept-item__checkbox--disabled': readonly
                      }"
                      @click.stop="handleDeptCheckboxToggle(dept)"></div>
                    <span class="dept-item__name">{{ dept.deptName || dept.dept }}</span>
                  </div>
                </div>
              </div>
            </template>
          </div>
        </div>
      </el-col>

      <el-col :span="12">
        <div class="side-card">
          <div class="side-card__header">
            <div class="header-left">
              {{ activeDeptInfo.dept || '部门人员列表' }}
            </div>
            <div class="header-right">
              <el-input v-model="staffSearchKeyword" class="header-search-input" size="mini" clearable
                placeholder="搜索用户名/姓名" @click.native.stop />
            </div>
          </div>
          <div class="side-card__body">
            <div class="table-wrapper" ref="tableWrapper">
              <template v-if="readonly">
                <el-table ref="staffTable" :data="filteredStaffList" border stripe :height="tableHeight"
                  v-loading="staffLoading" style="width: 100%;">
                  <el-table-column width="55" align="center">
                    <template slot-scope="scope">
                      <div class="readonly-check"
                        :class="{ 'readonly-check--checked': scope.row.selected === true }"></div>
                    </template>
                  </el-table-column>
                  <el-table-column type="index" label="序号" width="60" align="center" />
                  <el-table-column prop="username" label="用户名" width="150" />
                  <el-table-column prop="realname" label="姓名" min-width="120" />
                </el-table>
              </template>
              <template v-else>
                <el-table ref="staffTable" :data="filteredStaffList" border stripe :height="tableHeight"
                  v-loading="staffLoading" style="width: 100%;" @selection-change="handleSelectionChange">
                  <el-table-column type="selection" width="55"></el-table-column>
                  <el-table-column type="index" label="序号" width="60" align="center" />
                  <el-table-column prop="username" label="用户名" width="150" />
                  <el-table-column prop="realname" label="姓名" min-width="120" />
                </el-table>
              </template>
            </div>
          </div>
        </div>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import { receiverDeptList, receiverUserList } from './api'

export default {
  name: 'SelectDeptStaff',
  props: {
    taskInfo: {
      type: Object,
      'default': () => ({})
    },
    // 只读回显：禁用交互，仅展示当前已回显的选择状态
    readonly: {
      type: Boolean,
      'default': false
    }
  },
  data() {
    return {
      selectDept: {},
      partialStaffSelection: new Map(),
      currentStaffList: [],
      syncingTableSelection: false,
      deptStaffCache: new Map(),
      deletedUserMap: new Map(),
      lastInitKey: null,
      staffRequestSeq: 0,
      deptList: [],
      deptLoading: false,
      activeDeptId: null,
      staffLoading: false,
      tableHeight: 420,
      selectedStaffRows: [],
      deptSelectionState: {},
      activeDeptInfo: {
        dept: ''
      },
      staffSearchKeyword: '',
      deptSearchKeyword: '',
      receiverConfig: ''
    }
  },
  computed: {
    readonlySelectedDeptNames() {
      if (!this.readonly) return ''
      const names = []
      if (!Array.isArray(this.deptList)) return ''
      this.deptList.forEach((dept) => {
        const deptId = dept && dept.deptId
        if (!deptId) return
        const st = this.partialStaffSelection.get(deptId)
        if (!st || st.deptSelected !== true) return
        names.push(dept.deptName || dept.dept || String(deptId))
      })
      return names.join('、')
    },
    readonlyExcludedSummary() {
      if (!this.readonly) return ''
      const parts = []
      if (!Array.isArray(this.deptList)) return ''
      this.deptList.forEach((dept) => {
        const deptId = dept && dept.deptId
        if (!deptId) return
        const st = this.partialStaffSelection.get(deptId)
        if (!st || st.deptSelected !== true) return
        const list = this.deletedUserMap.get(String(deptId)) || []
        const usernames = list.map((u) => this.getUserKey(u)).filter(Boolean)
        if (!usernames.length) return
        const deptName = dept.deptName || dept.dept || String(deptId)
        parts.push(deptName + '：' + usernames.join('、'))
      })
      return parts.join('；')
    },
    filteredDeptList() {
      const list = this.deptList
      if (!Array.isArray(list)) return []

      const q = (this.deptSearchKeyword || '').trim().toLowerCase()
      if (!q) return list

      return list.filter((dept) => {
        const name = String(dept?.deptName ?? dept?.dept ?? '').toLowerCase()
        const id = String(dept?.deptId ?? '').toLowerCase()
        return name.includes(q) || id.includes(q)
      })
    },
    filteredStaffList() {
      const list = this.currentStaffList
      if (!Array.isArray(list)) return []
      const q = (this.staffSearchKeyword || '').trim().toLowerCase()
      if (!q) return list
      return list.filter((row) => {
        const u = String(row && row.username != null ? row.username : '').toLowerCase()
        const r = String(row && row.realname != null ? row.realname : '').toLowerCase()
        return u.includes(q) || r.includes(q)
      })
    },
    requestId() {
      const id = this.taskInfo && this.taskInfo.id
      if (id === null || id === undefined || id === '') {
        return null
      }
      return id
    }
  },
  watch: {
    staffSearchKeyword() {
      if (this.readonly) return
      this.syncingTableSelection = true
      this.$nextTick(() => {
        this.toggleSelection()
      })
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.updateTableHeight()
      window.addEventListener('resize', this.updateTableHeight)
    })
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.updateTableHeight)
  },
  methods: {
    syncReceiverConfig() {
      try {
        this.receiverConfig = JSON.stringify(this.getSelectionState())
      } catch (e) {
        // 忽略序列化异常
      }
    },
    reset() {
      this.deptList = []
      this.deptLoading = false
      this.activeDeptId = null
      this.staffLoading = false
      this.deptSelectionState = {}
      this.selectedStaffRows = []
      this.partialStaffSelection = new Map()
      this.currentStaffList = []
      this.syncingTableSelection = false
      this.deptStaffCache = new Map()
      this.deletedUserMap = new Map()
      this.lastInitKey = null
      this.staffRequestSeq = 0
      this.staffSearchKeyword = ''
      this.deptSearchKeyword = ''
      this.activeDeptInfo = { dept: '' }
      this.receiverConfig = ''
      this.tryInit()
    },

    tryInit() {
      const initKey = `${this.requestId || ''}`
      if (this.lastInitKey === initKey) {
        return
      }
      this.lastInitKey = initKey
      this.init()
    },

    init() {
      this.loadDeptData()
    },

    buildListParams(extra = {}) {
      const body = { ...extra }
      if (this.requestId != null && this.requestId !== '') {
        body.id = this.requestId
      }
      return body
    },

    async loadDeptData() {
      this.deptLoading = true
      try {
        const params = this.buildListParams()
        if (this.receiverConfig) {
          params.receiverConfig = this.receiverConfig
        }
        const res = await receiverDeptList(params)
        const deptList = Array.isArray(res && res.records)
          ? JSON.parse(JSON.stringify(res.records))
          : []

        this.deptList = deptList
        this.initDeptSelectionMap()
        this.activeDeptId = null
        this.currentStaffList = []
        this.activeDeptInfo = { dept: '' }
      } catch (e) {
        console.error('加载部门数据失败:', e)
        this.deptList = []
        this.currentStaffList = []
      } finally {
        this.deptLoading = false
      }
    },

    initDeptSelectionMap() {
      if (!Array.isArray(this.deptList)) return
      this.deptList.forEach(item => {
        const deptId = item && item.deptId
        if (!deptId) return
        const existing = this.partialStaffSelection.get(deptId)
        if (existing) return
        this.partialStaffSelection.set(deptId, {
          deptSelected: item.selected === true,
          staffList: []
        })
      })
      this.$forceUpdate()
    },

    buildCurrentReceiverConfig() {
      if (this.receiverConfig) return this.receiverConfig
      try {
        const state = this.getSelectionState()
        return JSON.stringify(state)
      } catch (e) {
        return ''
      }
    },

    async loadStaffData(deptId) {
      const params = this.buildListParams({
        departId: deptId,
        receiverConfig: this.buildCurrentReceiverConfig()
      })
      const res = await receiverUserList(params)
      return res && res.records
    },

    getUserKey(user) {
      if (!user || typeof user !== 'object') return ''
      return String(user.userBy || user.username || '')
    },

    buildDeletedUserItem(staff, dept) {
      const deptId = dept && dept.deptId
      const deptName = (dept && (dept.deptName || dept.dept || dept.belongDept)) || ''
      return {
        id: staff && staff.id ? staff.id : null,
        userBy: staff && (staff.userBy || staff.username) ? (staff.userBy || staff.username) : null,
        userByZh: staff && (staff.userByZh || staff.realname) ? (staff.userByZh || staff.realname) : null,
        belongDeptId: deptId ? String(deptId) : null,
        belongDept: deptName
      }
    },

    async handleSelectDept(dept) {
      if (!dept || !dept.deptId) return
      this.staffSearchKeyword = ''
      this.selectDept = dept
      const deptId = dept.deptId
      this.syncingTableSelection = true
      this.activeDeptId = deptId
      const requestSeq = ++this.staffRequestSeq

      this.staffLoading = true
      try {
        const res = await this.loadStaffData(deptId)
        const staffList = Array.isArray(res) ? res : []

        if (requestSeq !== this.staffRequestSeq) {
          this.syncingTableSelection = false
          return
        }

        this.deptStaffCache.set(deptId, staffList)

        const deptState = this.partialStaffSelection.get(deptId) || { deptSelected: false, staffList: [] }
        const hasSelectedStaff = staffList.some(s => s && s.selected === true)

        this.partialStaffSelection.set(deptId, {
          ...deptState,
          deptSelected: hasSelectedStaff || deptState.deptSelected,
          staffList
        })

        const deletedList = []
        if (hasSelectedStaff || deptState.deptSelected) {
          staffList.forEach(staff => {
            if (!staff || staff.selected === true) return
            deletedList.push(this.buildDeletedUserItem(staff, dept))
          })
        }
        this.deletedUserMap.set(String(deptId), deletedList)

        this.currentStaffList = staffList
        this.$nextTick(() => {
          this.toggleSelection()
        })
      } catch (e) {
        console.error('加载部门人员失败:', e)
        this.currentStaffList = []
        this.syncingTableSelection = false
      } finally {
        if (requestSeq === this.staffRequestSeq) {
          this.staffLoading = false
        }
      }

      const meta = this.deptList.find(item => item && item.deptId === dept.deptId)
      if (meta) {
        this.activeDeptInfo = {
          dept: meta.deptName || meta.dept || ''
        }
      }
    },

    toggleSelection() {
      if (this.readonly) {
        this.syncingTableSelection = false
        return
      }
      const table = this.$refs.staffTable
      const rows = this.filteredStaffList
      if (!table || !Array.isArray(rows)) {
        this.syncingTableSelection = false
        return
      }

      table.clearSelection()
      rows.forEach(row => {
        if (row.selected === true) {
          table.toggleRowSelection(row, true)
        }
      })

      this.$nextTick(() => {
        this.syncingTableSelection = false
      })
    },

    updateTableHeight() {
      const wrapper = this.$refs.tableWrapper
      if (!wrapper) return
      const computedHeight = wrapper.clientHeight
      if (computedHeight && computedHeight !== this.tableHeight) {
        this.tableHeight = computedHeight
      }
    },

    handleSelectionChange(rows) {
      if (this.readonly) return
      if (this.syncingTableSelection) return
      if (!this.activeDeptId) return
      const normalizedRows = Array.isArray(rows) ? rows : []
      const deptData = this.partialStaffSelection.get(this.activeDeptId)
      if (!deptData || !Array.isArray(deptData.staffList)) return

      const selectedUsernames = new Set(normalizedRows.map(row => this.getUserKey(row)).filter(Boolean))
      const visibleKeySet = new Set(this.filteredStaffList.map(row => this.getUserKey(row)).filter(Boolean))

      deptData.staffList.forEach(staff => {
        const key = this.getUserKey(staff)
        if (!key) return
        if (visibleKeySet.has(key)) {
          staff.selected = selectedUsernames.has(key)
        }
      })

      const hasSelectedStaff = deptData.staffList.some(staff => staff && staff.selected === true)
      this.partialStaffSelection.set(this.activeDeptId, {
        ...deptData,
        deptSelected: hasSelectedStaff,
        staffList: deptData.staffList
      })

      if (!hasSelectedStaff) {
        this.deletedUserMap.set(String(this.activeDeptId), [])
      } else {
        const dept = this.deptList.find(d => d && d.deptId === this.activeDeptId) || this.selectDept
        const prevList = this.deletedUserMap.get(String(this.activeDeptId)) || []
        const prevMap = new Map(prevList.map(item => [this.getUserKey(item), item]))
        const deletedList = []
        deptData.staffList.forEach(staff => {
          if (!staff || staff.selected === true) return
          const userKey = this.getUserKey(staff)
          if (!userKey) return
          const existing = prevMap.get(userKey)
          deletedList.push(existing || this.buildDeletedUserItem(staff, dept))
        })
        this.deletedUserMap.set(String(this.activeDeptId), deletedList)
      }

      this.syncReceiverConfig()
      this.$forceUpdate()
    },

    getDeptActiveStatus(deptId) {
      if (!deptId) return false
      const deptData = this.partialStaffSelection.get(deptId)
      return deptData && deptData.deptSelected === true
    },

    handleDeptCheckboxToggle(dept) {
      if (this.readonly) return
      if (!dept || !dept.deptId) return

      const existingData = this.partialStaffSelection.get(dept.deptId) || {}
      const currentSelected = existingData.deptSelected === true
      const newSelected = !currentSelected

      this.deletedUserMap.set(String(dept.deptId), [])

      const cachedStaffList = this.deptStaffCache.get(dept.deptId)
      if (Array.isArray(cachedStaffList)) {
        cachedStaffList.forEach(staff => {
          if (!staff) return
          staff.selected = newSelected === true
        })
      }

      this.partialStaffSelection.set(dept.deptId, {
        ...existingData,
        deptSelected: newSelected,
        staffList: Array.isArray(cachedStaffList) ? cachedStaffList : (existingData.staffList || [])
      })

      this.syncReceiverConfig()
      this.handleSelectDept(dept)
      this.$forceUpdate()
    },

    async prepareNoticeRuleEdit({ receiverConfig } = {}) {
      this.lastInitKey = null
      this.deptStaffCache.clear()
      this.partialStaffSelection = new Map()
      this.deletedUserMap = new Map()
      this.receiverConfig = receiverConfig || ''
      await this.loadDeptData()
      this.applyReceiverConfigState(receiverConfig)
      this.activeDeptId = null

      if (this.deptList.length) {
        const firstChecked = this.deptList.find((d) => {
          const st = d && this.partialStaffSelection.get(d.deptId)
          return st && st.deptSelected === true
        })
        await this.handleSelectDept(firstChecked || this.deptList[0])
      } else {
        this.currentStaffList = []
      }
    },

    applyReceiverConfigState(raw) {
      if (raw == null || raw === '') return
      let state = raw
      if (typeof raw === 'string') {
        try {
          state = JSON.parse(raw)
        } catch (e) {
          return
        }
      }
      if (!state || typeof state !== 'object') return

      // 只支持《人员组件前端对接文档》新结构字段：
      // { allEmployee, selectedDeptIds, excludedUsersByDept }
      const allEmployee = state.allEmployee === true || state.allEmployee === 1 || state.allEmployee === '1' || String(state.allEmployee).toLowerCase() === 'true'
      const selectedDeptIds = Array.isArray(state.selectedDeptIds) ? state.selectedDeptIds.map(String) : []
      const excludedUsersByDeptRaw = (state.excludedUsersByDept && typeof state.excludedUsersByDept === 'object')
        ? state.excludedUsersByDept
        : {}

      const nextDeleted = new Map()
      Object.keys(excludedUsersByDeptRaw).forEach((deptId) => {
        const did = deptId != null ? String(deptId) : ''
        if (!did) return

        const list = excludedUsersByDeptRaw[deptId]
        if (!Array.isArray(list)) return

        const normalizedUsernames = list
          .map((u) => {
            if (!u) return null
            if (typeof u === 'string') return u
            if (typeof u === 'object') return u.userBy || u.username || null
            return null
          })
          .filter(Boolean)

        const deletedList = normalizedUsernames.map((username) => ({
          // 复用组件内部 getUserKey()：userBy || username
          userBy: username,
          username,
          belongDeptId: did
        }))
        nextDeleted.set(did, deletedList)
      })

      this.deletedUserMap = nextDeleted

      if (!Array.isArray(this.deptList)) return
      this.deptList.forEach((dept) => {
        const deptId = dept && dept.deptId
        if (!deptId) return

        const idStr = String(deptId)
        const deptSelected = allEmployee ? true : selectedDeptIds.includes(idStr)

        const prev = this.partialStaffSelection.get(deptId) || {}
        this.partialStaffSelection.set(deptId, {
          ...prev,
          deptSelected,
          staffList: Array.isArray(prev.staffList) ? prev.staffList : []
        })
      })

      this.$forceUpdate()
    },

    getSelectionState() {
      // 输出字段对齐《人员组件前端对接文档》：
      // {
      //   allEmployee: false,
      //   selectedDeptIds: ['deptA', 'deptB'],
      //   excludedUsersByDept: {
      //     deptA: ['username1', 'username2'],
      //     deptB: []
      //   }
      // }
      const selectedDeptIds = []
      const excludedUsersByDept = {}

      this.partialStaffSelection.forEach((item, key) => {
        if (!item || item.deptSelected !== true) return

        const deptId = key != null ? String(key) : ''
        if (!deptId) return

        selectedDeptIds.push(deptId)

        const deletedList = this.deletedUserMap.get(deptId) || []
        const excludedUsernames = deletedList
          .map((staff) => this.getUserKey(staff))
          .filter(Boolean)

        excludedUsersByDept[deptId] = excludedUsernames
      })

      return {
        allEmployee: false,
        selectedDeptIds,
        excludedUsersByDept
      }
    }
  }
}
</script>

<style scoped lang="less">
.dept-staff-panel {
  padding: 10px 0;

  ::v-deep .el-row {
    flex-wrap: nowrap;
  }

  ::v-deep .el-col {
    min-width: 0;
  }

  .header-search-input {
    width: 132px;
  }

  .dept-checkbox-group {
    display: flex;
    flex-direction: column;
  }

  .side-card {
    border: 1px solid #e8eaec;
    border-radius: 4px;
    background: #fff;
    display: flex;
    flex-direction: column;
    height: 50vh;

    &__header {
      padding: 12px 16px;
      border-bottom: 1px solid #e8eaec;
      font-weight: 600;
      font-size: 14px;
      color: #515a6e;
      display: flex;
      align-items: center;
      gap: 12px;
      flex-wrap: wrap;

      .header-left {
        font-weight: 600;
        flex: 1;
        min-width: 0;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }

      .header-right {
        display: flex;
        align-items: center;
        flex-shrink: 0;
        margin-left: auto;
        gap: 10px;
        font-weight: normal;
        font-size: 13px;
        color: #606266;
      }
    }

    &__body {
      flex: 1;
      overflow-y: auto;
      overflow-x: auto;

      .table-wrapper {
        height: 100%;
        min-width: 0;
      }
    }

    &__placeholder {
      text-align: center;
      color: #999;
      padding: 60px 0;
      font-size: 14px;
    }
  }

  .dept-item {
    padding: 15px 10px;
    border-bottom: 1px solid #dcdcdc;
    cursor: pointer;
    transition: background-color 0.2s, color 0.2s;

    &__row {
      display: flex;
      align-items: center;
      gap: 8px;
    }

    &__name {
      flex: 1;
      min-width: 0;
      font-size: 14px;
      font-weight: 600;
      margin-bottom: 6px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }

    &__stats {
      font-size: 14px;
      color: #666;
      flex-shrink: 0;
      white-space: nowrap;
      text-align: right;
      min-width: 60px;
      margin-left: auto;
    }

    &__checkbox {
      margin-right: 4px;
      width: 16px;
      height: 16px;
      border-radius: 2px;
      border: 1px solid #dcdfe6;
      box-sizing: border-box;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      background-color: #fff;
      transition: all 0.2s;
      flex-shrink: 0;

      &::after {
        content: '';
        width: 8px;
        height: 4px;
        border-left: 2px solid transparent;
        border-bottom: 2px solid transparent;
        transform: rotate(-45deg) translateY(-1px);
        transition: border-color 0.2s;
      }
    }

    &__checkbox--checked {
      background-color: var(--theme-color, #546afc);
      border-color: var(--theme-color, #546afc);

      &::after {
        border-color: #fff;
      }
    }

    &__checkbox--disabled {
      cursor: not-allowed;
      opacity: 0.6;
      pointer-events: none;
    }

    &--active {
      background: color-mix(in srgb, var(--theme-color) 18%, transparent);

      .dept-item__name,
      .dept-item__stats {
        color: var(--theme-color);
        font-weight: 600;
      }
    }

    &:hover {
      background: color-mix(in srgb, var(--theme-color) 12%, transparent);

      .dept-item__name,
      .dept-item__stats {
        color: var(--theme-color);
      }
    }
  }

  .table-empty {
    text-align: center;
    padding: 40px 0;
    color: #999;
  }

  .readonly-check {
    width: 16px;
    height: 16px;
    border-radius: 2px;
    border: 1px solid #dcdfe6;
    box-sizing: border-box;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background-color: #fff;
    cursor: not-allowed;
    opacity: 0.6;
    margin: 0 auto;

    &::after {
      content: '';
      width: 8px;
      height: 4px;
      border-left: 2px solid transparent;
      border-bottom: 2px solid transparent;
      transform: rotate(-45deg) translateY(-1px);
    }

    &--checked {
      background-color: var(--theme-color, #546afc);
      border-color: var(--theme-color, #546afc);

      &::after {
        border-color: #fff;
      }
    }
  }
}

.readonly-summary {
  border: 1px solid #e8eaec;
  border-radius: 4px;
  background: #fff;
  padding: 10px 12px;
  margin-bottom: 12px;
  color: #606266;

  &__title {
    font-weight: 600;
    color: #515a6e;
    margin-bottom: 8px;
  }

  &__row {
    display: flex;
    gap: 8px;
    line-height: 20px;
    margin-top: 4px;
  }

  &__label {
    flex-shrink: 0;
  }

  &__value {
    flex: 1;
    min-width: 0;
    word-break: break-all;
    color: #303133;
  }
}

</style>
