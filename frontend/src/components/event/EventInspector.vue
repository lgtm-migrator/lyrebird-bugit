<template>
  <div>
    <Table
      ref="eventTable"
      size="small"
      highlight-row
      :columns="columns"
      :data="events"
      @on-filter-change="onFilterChange"
      @on-row-click="onRowClick"
      style="height:100%"
      class="event-table"
    >
      <template slot-scope="{ row }" slot="action">
        <ContextMenuItem>
          <Button type="text" shape="circle" icon="ios-add-circle-outline" @click="onAddEvent(row)"></Button>
        </ContextMenuItem>
      </template>
      <template slot-scope="{ row }" slot="timestamp">
        <ContextMenuItem>
          <div>{{ts2String(row.timestamp)}}</div>
        </ContextMenuItem>
      </template>
      <template slot-scope="{ row }" slot="channel">
        <ChannelColumn :channel="row.channel"></ChannelColumn>
      </template>
      <template slot-scope="{ row }" slot="content">
        <component :is="contentComponentName(row.content)" :event="content2Obj(row.content)"></component>
      </template>
    </Table>
    <div class="page">
      <Page
        v-if="page"
        :current="page.index+1"
        :total="page.count*page.size"
        :page-size="page.size"
        @on-change="onPageChange"
      ></Page>
    </div>
    <Spin fix v-if="loadingEvents">Loading Events...</Spin>
    <div
      v-show="isContextMenuShown"
      class="row-contextmenu ivu-select-dropdown"
      :style="{left:contextMenuLeft, top:contextMenuTop}"
    >
      <ul class="ivu-dropdown-menu">
        <li class="ivu-dropdown-item" @click="onContextMenuShowNotice">Show notice</li>
        <li class="ivu-dropdown-item" @click="onContextMenuShowAll">Show all</li>
      </ul>
    </div>
  </div>
</template>

<script>
import FlowTableItem from '@/components/event/FlowTableItem.vue'
import CustomTableItem from '@/components/event/CustomTableItem.vue'
import ChannelColumn from '@/components/event/ChannelColumn.vue'
import ContextMenuItem from '@/components/event/ContextMenuItem.vue'

export default {
  components: {
    FlowTableItem,
    CustomTableItem,
    ChannelColumn,
    ContextMenuItem
  },
  created () {
    this.$store.dispatch('loadChannelNames')
    const urlParams = new URLSearchParams(window.location.search)
    const eventId = urlParams.get('event_id')
    this.$store.dispatch('loadEvents', { eventId: eventId })
    this.$bus.$on('contextmenu.show', this.showContextMenu)
    this.$bus.$on('contextmenu.dismiss', this.dismissContextMenu)
  },
  mounted () {
    this.tableRect = this.$refs.eventTable.$el.getBoundingClientRect()
  },
  data () {
    return {
      tableRect: null,
      isContextMenuShown: false,
      contextMenuLeft: 0,
      contextMenuTop: 0,
      refreshEventListTimer: null
    }
  },
  computed: {
    events () {
      return this.$store.state.event.events
    },
    loadingEvents () {
      return this.$store.state.event.loadingEvents
    },
    page () {
      return this.$store.state.event.page
    },
    channelNames () {
      return this.$store.state.event.channelNames
    },
    columns () {
      let filters = []
      for (const channelName of this.$store.state.event.channelNames) {
        filters.push({ label: channelName, value: channelName })
      }
      return [
        {
          title: '',
          slot: 'action',
          width: 35
        },
        {
          title: 'Time',
          key: 'timestamp',
          slot: 'timestamp',
          width: 90
        },
        {
          title: 'Channel',
          key: 'channel',
          slot: 'channel',
          width: 80,
          filters: filters,
          filteredValue: this.$store.state.event.channelFilters,
          filterRemote (value) {
            this.$store.dispatch('updateChannelFilters', value)
          }
        },
        {
          title: 'Content',
          key: 'content',
          slot: 'content'
        }
      ]
    },
    searchStr () {
      return this.$store.state.event.searchStr
    }
  },
  watch: {
    searchStr (newValue, oldValue) {
      if (newValue === null) {
        newValue = ''
      }
      clearTimeout(this.refreshEventListTimer)
      this.refreshEventListTimer = setTimeout(() => {
        if (newValue.trim() !== oldValue.trim()) {
          this.$store.dispatch('loadEvents')
          clearTimeout(this.refreshEventListTimer)
        }
      }, 500)
    }
  },
  methods: {
    content2Obj (content) {
      return JSON.parse(content)
    },
    contentComponentName (content) {
      const eventObj = JSON.parse(content)
      if (eventObj.channel === 'flow') {
        return 'FlowTableItem'
      } else {
        return 'CustomTableItem'
      }
    },
    onFilterChange (opt) {
      console.log('filter change', opt)
    },
    onRowClick (row) {
      this.$store.commit('setSelectedEventId', row.event_id)
      const prettyJson = JSON.stringify(JSON.parse(row.content), null, 2)
      this.$store.commit('setEventDetail', prettyJson)
      this.isContextMenuShown = false
    },
    ts2String (timeStamp) {
      let date = new Date(timeStamp * 1000)
      let hour = date.getHours() + ':'
      let minute = (date.getMinutes() < 10 ? '0' + date.getMinutes() : date.getMinutes()) + ':'
      let second = date.getSeconds() < 10 ? '0' + date.getSeconds() : date.getSeconds()
      return date.getMonth() + 1 + '-' + date.getDate() + ' ' + hour + minute + second
    },
    onPageChange (page) {
      this.$store.commit('setSelectedEventId', null)
      this.$store.commit('setEventDetail', '')
      this.$store.dispatch('loadEvents', { page: page - 1 })
    },
    onAddEvent (row) {
      const eventObj = JSON.parse(row.content)
      if (eventObj.summary) {
        this.$bus.$emit('addSummary', eventObj.summary)
      }
      if (eventObj.message) {
        this.$bus.$emit('addMessage', {
          channel: eventObj.channel,
          message: eventObj.message,
          id: eventObj.id,
          sender: eventObj.sender
        })
      }
      if (eventObj.snapshot) {
        let fileName = `snapshot_${eventObj.id}`
        this.$bus.$emit('addExportAttachments', {
          attachmentName: fileName,
          attachmentObj: eventObj,
          attachmentType: 'lb'
        })
      }
      if (eventObj.export) {
        // Export to attachment with a specified type.
        if (eventObj.export.hasOwnProperty('converter')) {
          let fileName = `${eventObj.channel}_${eventObj.id}`
          this.$bus.$emit('addExportAttachments', {
            attachmentName: fileName,
            attachmentObj: eventObj,
            // TODO: How to get other type's of file
            attachmentType: 'json'
          })
        }
      }
    },
    showContextMenu (event) {
      const rect = this.$refs.eventTable.$el.getBoundingClientRect()
      this.contextMenuLeft = event.pageX - rect.left + 'px'
      this.contextMenuTop = event.pageY - this.tableRect.top + 'px'
      this.isContextMenuShown = true
    },
    dismissContextMenu () {
      this.isContextMenuShown = false
    },
    onContextMenuShowNotice () {
      this.$store.dispatch('showNotice')
      this.isContextMenuShown = false
    },
    onContextMenuShowAll () {
      this.$store.dispatch('showAll')
      this.isContextMenuShown = false
    }
  }
}
</script>

<style less>
.page {
  text-align: center;
  margin-top: 5px;
}
.event-table .ivu-table-cell {
  padding-left: 3px;
  padding-right: 3px;
}
.row-contextmenu {
  position: absolute;
}
</style>
