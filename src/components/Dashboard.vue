<template lang='pug'>

  transition-group(name='fade').widget.dashboard
    .info-container(key='info')
      .name {{ user ? user.name : '加载中' }}
      .identity {{ user ? user.identity : '…' }}
        router-link(v-if='user.isNewbie' to='/intro') 新生指引 ＞
      img.icon.grayscale(@click='tidyMode = !tidyMode' :src='tidyMode ? expandImg : collapseImg')
      router-link(to='/download')
        img.icon(:src='downloadImg')
      img.icon(@click='logout()' :src='logoutImg')
    banner(key='banner' v-show='!tidyMode')

    .dashboard-container.border-top(key='dashboard' v-if='user')

      .row
        item(:icon='iconCard' name='余额' :value='card && card.info && card.info.balance' :is-stale='card && card.isStale' route='/card')
        item(v-if='isUndergraduate' :icon='iconLecture' name='人文讲座' :value='lecture && lecture.length' :is-stale='lecture && lecture.isStale' route='/lecture')

      .row
        item(v-if='isUndergraduate' :icon='iconSrtp' name='SRTP' :value='srtp && srtp.info.points' :is-stale='srtp && srtp.isStale' route='/srtp')
        item(v-if='isStudent' :icon='iconGrade' :name='(isGraduate ? "成绩" : "总绩点")' :value='gpa && (gpa.gpa || gpa.score || "暂无")' :is-stale='gpa && gpa.isStale' route='/grade' :is-graduate='isGraduate')

      .row(v-if='!tidyMode')
        item(v-if='isUndergraduate' name='跑操' :value='pe && pe.count' :is-stale='pe && pe.isStale' route='/pe')
        item(name='借书' :value='library && library.length' :is-stale='library && library.isStale' route='/library')
        item(v-if='isUndergraduate' name='CET' route='/cet' value='›')

      .row(v-if='!tidyMode')
        item(name='失物招领&寻物启事' route='/lost-and-found' :value='lostAndFoundMsg')

      //- .row(v-if='!tidyMode')
      //-   item(name='自定义考试提醒' route='/custom-exam' value='›')
    
      .row(v-if='!tidyMode')
        item(name='校历' route='/schedule' value='›')
        item(name='校车' route='/bus' value='›')
        item(name='空教室' route='/classroom' value='›')
        //- item(name='洗衣房' route='/laundry' value='›')
        //- item(name='App' route='/download' value='›')

      .row(v-if='!tidyMode && user.admin && user.admin.maintenance')
        item(name='系统概况' route='/admin/monitor' value='›')
        item(name='权限管理' route='/admin/privilege' value='›')
        item(name='通知管理' route='/admin/notice' value='›')

      .row(v-if='!tidyMode && user.admin && user.admin.publicity')
        item(name='轮播管理' route='/admin/banner' value='›')
        item(name='活动管理' route='/admin/activity' value='›')

      .row
        item(name='通知' route='/notice' :value='curNotice && curNotice.title || "暂无通知"')

</template>

<script>
import item from "./DashboardItem.vue";
import api from "@/api";
import downloadImg from "static/images/download.png";
import logoutImg from "static/images/logout.png";
import collapseImg from "static/images/collapse.png";
import expandImg from "static/images/expand.png";
import iconCard from "static/images/card.svg";
import iconLecture from "static/images/lecture.svg";
import iconSrtp from "static/images/srtp.svg";
import iconGrade from "static/images/grade.svg";
import banner from "./Banner.vue";
import moment from "moment";

export default {
  props: ["user"],
  components: {
    item,
    banner
  },
  data() {
    return {
      pe: null,
      gpa: null,
      card: null,
      srtp: null,
      lecture: null,
      library: null,
      notice: null,
      activities: null,
      curNoticeIndex: 0,
      downloadImg,
      logoutImg,
      collapseImg,
      expandImg,
      iconCard,
      iconLecture,
      iconSrtp,
      iconGrade,
      tidyMode: false,
      lostAndFoundMsg: "›",
    };
  },
  persist: {
    pe: "herald-default-pe",
    gpa: "herald-default-gpa",
    card: "herald-default-card",
    srtp: "herald-default-srtp",
    lecture: "herald-default-lecture",
    library: "herald-default-library",
    notice: "herald-default-notice",
    activities: "herald-default-activities"
  },
  async created() {
    // 下列不能用 await，否则前面的语句会阻塞后面的语句，前面的异常会阻止后面的语句
    api.get("/api/pe").then(res => (this.pe = res));
    api.get("/api/gpa").then(res => (this.gpa = res));
    api.get("/api/card").then(res => (this.card = res));
    api.get("/api/srtp").then(res => (this.srtp = res));
    api.get("/api/lecture").then(res => (this.lecture = res));
    api.get("/api/library").then(res => (this.library = res));
    api.get("/api/activity?pagesize=20").then(res => (this.activities = res));
    api.get("/api/notice").then(res => {
      this.notice = res;
      setInterval(() => {
        this.curNoticeIndex = this.filteredNotice.length
          ? (this.curNoticeIndex + 1) % this.filteredNotice.length
          : 0;
      }, 5000);
    });
    api.get("/api/lostAndFound/message").then(res => {
      let amount = 0;
      Object.keys(res).forEach(id => {
        amount += res[id];
      });
      if (amount) {
        this.lostAndFoundMsg = amount + "条新消息";
      } else {
        this.lostAndFoundMsg = "›";
      }
    });
  },
  methods: {
    logout() {
      this.$store.commit("logout");
    }
  },
  computed: {
    isStudent() {
      return this.user && /生/.test(this.user.identity);
    },
    isUndergraduate() {
      return this.user && /本科/.test(this.user.identity);
    },
    isGraduate() {
      return this.isStudent && !this.isUndergraduate;
    },
    filteredNotice() {
      // 这里过滤两天内的最近通知
      return (
        this.notice &&
        this.notice.filter(k => {
          return (
            k.time >=
            +moment()
              .startOf("day")
              .subtract(1, "day")
          );
        })
      );
    },
    curNotice() {
      return this.filteredNotice && this.filteredNotice[this.curNoticeIndex];
    }
  }
};
</script>

<style lang="stylus" scoped>
.widget.dashboard {
  overflow: hidden;

  .fade-enter-active, .fade-leave-active, .fade-move {
    overflow: hidden;
    max-height: 100vh;
    transition: max-height 0.3s, opacity 0.3s !important;
    position: relative;
  }

  .fade-enter, .fade-leave-to {
    opacity: 0 !important;
    max-height: 0 !important;
  }

  .info-container {
    padding: 0 0 15px;
    display: flex;
    flex-direction: row;
    align-items: center;

    * + * {
      margin-left: 10px;
    }

    .name {
      font-size: 17px;
      flex: 0 0 auto;
      white-space: nowrap;
      color: var(--color-text-regular);
    }

    .identity {
      font-size: 12px;
      flex: 1 1 0;
      white-space: nowrap;
      color: var(--color-text-secondary);

      a {
        margin-left: 5px;
      }
    }

    .text-icon {
      display: flex;
      flex-direction: row;
      align-items: center;

      .text {
        margin-left: 3px;
        color: #777;
      }
    }

    .icon {
      width: 20px;
      height: 20px;
      cursor: pointer;

      &.grayscale {
        filter: grayscale();
      }
    }
  }

  .dashboard-container {
    display: flex;
    flex-direction: column;
    box-sizing: border-box;
    margin: 5px -20px -15px;

    .row {
      display: flex;
      flex-direction: row;

      > * {
        flex: 1 1 0;
      }
    }
  }

  .admin-container {
    padding: 0 0 15px;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;

    .subtitle {
      font-size: 12px;
      color: var(--color-text-secondary);
      flex: 1 1 0;
    }

    .function {
      color: #333;
    }

    a + a .function {
      padding-left: 7px;
      border-left: 0.5px solid var(--color-divider);
      margin-left: 7px;
    }
  }
}
</style>
