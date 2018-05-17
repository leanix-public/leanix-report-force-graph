<template>
  <div id="app">
    <div class="title-box">
      <span class="business-capabilities">Business Capabilities</span>
      and
      <span class="applications">Applications</span>
    </div>
    <d3-network
      :net-nodes="nodes"
      :net-links="links"
      :options="options"
      :nodeCb="nodeFormatter"
      :linkCb="linkFormatter"
      @node-click="handleNodeClicked"
      />
  </div>
</template>

<script>
import D3Network from 'vue-d3-network'

export default {
  name: 'App',
  components: {
    D3Network
  },
  data () {
    return {
      width: 600,
      height: 600,
      // local cache for leanix data
      allFactSheets: [],
      nodes: [
      ],
      links: [
      ],
      nodeSize: 30
    }
  },
  computed: {
    options () {
      return {
        force: 5000,
        size: {
          w: this.width,
          h: this.height
        },
        nodeSize: this.nodeSize,
        nodeLabels: true,
        linkWidth: 3,
        strLinks: true
      }
    }
  },
  methods: {
    nodeFormatter: node => {
      switch (node.type) {
        case 'Application':
          node._color = '#E91E63'
          // node._size = 20
          break
        case 'BusinessCapability':
          node._color = '#4b1ac3'
          node._size = node.level ? 50 - node.level * 7 : 30
          break
      }
      node._labelClass = 'node-label'
      return node
    },
    linkFormatter: link => {
      link = Object.assign(link, { _color: '#BDBDBD' })
      return link
    },
    handleResize: function (evt) {
      this.width = evt.target.innerWidth - 40
      this.height = evt.target.innerHeight - 24
    },
    handleNodeClicked (evt, node) {
      node.expanded = !node.expanded
      this.toggleNodeChildren(node)
    },
    toggleNodeChildren (node) {
      let children = []
      if (node.type === 'BusinessCapability') {
        children = node.relToChild.edges
          .map(edge => edge.node.factSheet.id)
        const appIDs = node.relBusinessCapabilityToApplication.edges.map(edge => edge.node.factSheet.id)
        children = children.concat(appIDs)
      }
      const getChildrenIDs = (factSheet, ids) => {
        if (ids === undefined) ids = []
        if (factSheet.type === 'BusinessCapability') {
          factSheet.relBusinessCapabilityToApplication.edges
            .map(edge => edge.node.factSheet)
            .forEach(childFactSheet => {
              ids.push(childFactSheet.id)
            })
          factSheet.relToChild.edges
            .map(edge => edge.node.factSheet)
            .forEach(childFactSheet => {
              ids.push(childFactSheet.id)
              const factSheet = this.allFactSheets[childFactSheet.id]
              ids = getChildrenIDs(factSheet, ids)
            })
        }
        return ids
      }

      const nodesIdx = this.nodes.map(node => node.id)

      const allChildren = getChildrenIDs(this.allFactSheets[node.id])
      const allChildrenFactSheets = allChildren.map(id => this.allFactSheets[id])

      const allChildrenFactSheetsIdx = allChildrenFactSheets
        .map(factSheet => nodesIdx.indexOf(factSheet.id))

      const childrenFactSheets = children.map(id => this.allFactSheets[id])

      const childrenFactSheetsIdx = childrenFactSheets
        .map(factSheet => nodesIdx.indexOf(factSheet.id))

      if (node.expanded) {
        const inserts = childrenFactSheetsIdx.filter(idx => idx < 0)
          .map((idx, i) => childrenFactSheets[i])
        const links = inserts
          .map(factSheet => { return { sid: node.id, tid: factSheet.id, _color: '#7986CB' } })
        this.nodes = this.nodes.concat(inserts)
        this.links = this.links.concat(links)
      } else {
        const allSources = [node.id, ...allChildren]
        this.links = this.links.filter(link => allSources.indexOf(link.sid) < 0)
        allChildrenFactSheetsIdx
          .filter(idx => idx > -1)
          .sort()
          .reverse()
          .forEach(idx => this.nodes.splice(idx, 1))
      }
    },
    fetchAllBusinessCapabilities () {
      const query = `
      query($filter:FilterInput){
        allFactSheets(filter:$filter) {
          edges {
            node {
              id
              name
              type
              level
              ...on BusinessCapability {
                relToChild {edges{node{factSheet{id}}}}
                relToParent {edges{node{factSheet{id}}}}
                relBusinessCapabilityToApplication{edges{node{factSheet{id}}}}
              }
              ...on Application {
                relApplicationToBusinessCapability{edges{node{factSheet{id}}}}
              }
            }
          }
        }
      }`
      const variables = {
        filter: {
          facetFilters: [
            { facetKey: 'FactSheetTypes', keys: ['BusinessCapability', 'Application'] }
          ]
        }
      }
      this.$lx.executeGraphQL(query, variables)
        .then(res => {
          this.allFactSheets = res.allFactSheets.edges
            .map(edge => edge.node)
            .reduce((accumulator, node) => {
              accumulator[node.id] = node
              return accumulator
            }, {})
          this.nodes = JSON.parse(JSON.stringify(Object.values(this.allFactSheets)))
            .filter(factSheet => factSheet.type === 'BusinessCapability' && factSheet.level === 1)
        })
        .catch(err => {
          console.log(err)
        })
    }
  },
  mounted () {
    window.addEventListener('resize', this.handleResize)
    this.$lx.init()
      .then(setup => {
        this.$lx.ready({})
      })
    this.fetchAllBusinessCapabilities()
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.handleResize)
  }
}
</script>

<style>

</style>

<style lang="stylus">
  @import './stylus/main'
  @import './stylus/material-shadows'
  #app
    z-depth-2dp()
    display flex
    flex-flow column
    justify-content center
    align-items center
    height calc(100vh - 24px)
    position relative
  .title-box
    position absolute
    top 0
    left 0
    padding 1rem
    font-size 2rem
    .business-capabilities
      font-weight bold
      color #4b1ac3
    .applications
      font-weight bold
      color #E91E63

  .node-label
    font-size: 0.75rem
</style>
