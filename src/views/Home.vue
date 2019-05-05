<template>
  <div class="home">
    <kanban-board :stages="stages" :blocks="blocks"></kanban-board>
  </div>
</template>

<script>
import KanbanBoard from '@/components/KanbanBoard'
export default {
  name: 'home',
  data () {
    return {
      stages: ['on-hold', 'in-progress', 'needs-review', 'approved'],
      blocks: [
        {
          id: 1,
          status: 'on-hold',
          title: 'Test'
        }
      ]
    }
  },
  components: {
    KanbanBoard
  }
}
</script>
<style lang='stylus' rel='stylesheet/stylus'>
$on-hold = #FB7D44;
$in-progress = #2A92BF;
$needs-review = #F4CE46;
$approved = #00B961;
* {
  box-sizing: border-box;
}
body {
  background: #33363D;
  color: white;
  font-family: 'Lato';
  font-weight: 300;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}
.drag-column {
  .drag-column-header > div {
    width: 100%;
    h2 > a {
      float: right;
    }
  }
  .drag-column-footer > div {
      margin-left: 10px;
      a {
          text-decoration: none;
          color: white;
          &:hover {
              text-decoration: underline;
          }
      }
  }
  &-on-hold {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $on-hold;
    }
  }
  &-in-progress {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $in-progress;
    }
  }
  &-needs-review {
    .drag-column-header,
    .is-moved,
    .drag-options{
      background: $needs-review;
    }
  }
  &-approved {
    .drag-column-header,
    .is-moved,
    .drag-options {
      background: $approved;
    }
  }
}
.section {
  padding: 20px;
  text-align: center;
  a {
    color: white;
    text-decoration: none;
    font-weight: 300;
  }
  h4 {
    font-weight: 400;
    a {
      font-weight: 600;
    }
  }
}
</style>
