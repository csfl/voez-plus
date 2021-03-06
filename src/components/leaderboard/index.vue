<template>
  <div class="voez-leaderboard">
    <div class="voez-leaderboard-song-info">
      <p>{{ song.author }}</p>
      <p>BPM {{ song.bpm }}</p>
    </div>
    <div class="voez-leaderboard-song-cover" @click="toggleCoverBlur">
      <div class="vplus-leaderboard-song-cover-box" v-show="selfTop && cover.blur" transition="fade">
        <blur :blur-amount="5" :url="coverSrc" :height="cover.height">
          <div style="width: 100%; height: 100%" v-if="selfTop">
            <transition name="fade">
              <div
                class="vplus-leaderboard-self"
                v-if="selfTop.rank !== undefined"
              >
                <template v-if="selfTop.rank > 0">
                  <p>{{ selfTop.name }}</p>
                  <p>Score: {{ selfTop.score }}</p>
                  <p>Rank: {{ selfTop.rank }}</p>
                </template>
                <p v-else>没有记录</p>
              </div>
              <div class="vplus-leaderboard-self-notes" v-if="selfTop.notes" transition="fade">
                <div class="vplus-leaderboard-self-note-square mp">{{ selfTop.notes.mp }}</div>
                <div class="vplus-leaderboard-self-note-square perfect">{{ selfTop.notes.perfect }}</div>
                <div class="vplus-leaderboard-self-note-square combo">{{ selfTop.notes.combo }}</div>
                <div class="vplus-leaderboard-self-note-square ok">{{ selfTop.notes.ok }}</div>
                <div class="vplus-leaderboard-self-note-square miss">{{ selfTop.notes.miss }}</div>
                <div class="vplus-leaderboard-self-grade" :class="{ ap: selfTop.ap, fc: selfTop.fc }"
                     v-if="selfTop.notes">
                  {{ selfGrade }}
                  <div class="vplus-leaderboard-self-grade-bg"></div>
                </div>
              </div>
            </transition>
          </div>
        </blur>
      </div>
      <img :src="coverSrc" :alt="song.name">
    </div>
    <sticky>
      <tab :active-color="modeColor">
        <tab-item :selected="selectedMode === 'easy'" @on-item-click="selectedMode = 'easy'">Easy {{ song.level_easy }}
          <spinner type="dots" v-show="!tops.easy"></spinner>
        </tab-item>
        <tab-item :selected="selectedMode === 'hard'" @on-item-click="selectedMode = 'hard'">Hard {{ song.level_hard }}
          <spinner type="dots" v-show="!tops.hard"></spinner>
        </tab-item>
        <tab-item :selected="selectedMode === 'special'" @on-item-click="selectedMode = 'special'">Special {{
          song.level_special }}
          <spinner type="dots" v-show="!tops.special"></spinner>
        </tab-item>
      </tab>
    </sticky>
    <template v-if="leaderboard">
      <divider v-show="!leaderboard.length" style="margin-top: 15px">暂无记录</divider>
      <group :title="song.name" v-show="leaderboard.length > 0">
        <leaderboard-item
          v-for="(player, index) of leaderboard"
          :key="index"
          :item="player"
          :is-self="selfTop && selfTop.rank === index + 1"
          :rank="index + 1"
        />
      </group>
    </template>
  </div>
</template>
<script>
  import {mapGetters} from 'vuex'
  import {Sticky, Tab, TabItem, Blur, Group, Cell, Spinner, Divider} from 'vux'
  import LeaderboardItem from './leaderboard-item.vue'
  import {axios} from '../../helpers/request'

  const MODES = ['easy', 'hard', 'special']

  const iPad = /ipad/.test(window.navigator.userAgent.toLocaleLowerCase())
  const pro = iPad && document.body.clientWidth >= 1024 && document.body.clientHeight >= 1024
  export default {
    components: {
      Sticky,
      Tab,
      TabItem,
      Group,
      Cell,
      Blur,
      LeaderboardItem,
      Spinner,
      Divider
    },
    data () {
      return {
        selectedMode: 'special',
        tops: {
          easy: null,
          hard: null,
          special: null
        },
        self: {
          easy: {},
          hard: {},
          special: {}
        },
        displayMode: 'score',
        cover: {
          height: iPad ? ((window.innerWidth - (pro ? 375 : 320) - 60) / 2) : window.innerWidth * 0.5625,
          blur: true
        }
      }
    },
    computed: {
      ...mapGetters([
        'songMap'
      ]),
      songId () {
        return +this.$route.params.songId
      },
      song () {
        return this.songMap[this.songId] || {}
      },
      coverSrc () {
        if (!this.songId) {
          return 'http://voez.sevenoutman.com/voez/cover/default/750'
        }
        return 'http://voez.sevenoutman.com/voez/cover/' + this.songId + '/750'
      },
      modeColor () {
        if (this.selectedMode === 'special') {
          return '#F09412'
        }
        if (this.selectedMode === 'hard') {
          return '#E13841'
        }
        if (this.selectedMode === 'easy') {
          return '#169CBF'
        }
        return '#04be02'
      },
      leaderboard () {
        return this.tops[this.selectedMode]
      },
      selfTop () {
        return this.self[this.selectedMode]
      },
      hasSelfTop () {
        return Object.keys(this.selfTop).length > 0
      },
      selfGrade () {
        if (!this.selfTop) {
          return ''
        }
        if (this.selfTop.fc) {
          return 'S'
        }
        if (this.selfTop.score >= 950000) {
          return 'S'
        }
        if (this.selfTop.score >= 880000) {
          return 'A'
        }
        if (this.selfTop.score >= 800000) {
          return 'B'
        }
        return 'C'
      }
    },
    methods: {
      toggleCoverBlur () {
        this.cover.blur = !this.cover.blur
      }
    },
    beforeRouteEnter (_, __, next) {
      next(vm => {
        document.title = vm.song.name
      })
    },
    mounted (to, from, next) {
      let songId = this.$route.params.songId

      MODES.forEach(mode => {
        axios(`voez/leaderboard/${songId}/${mode}`).then(({data}) => {
          let res = data
          if (res.result && res.code === 0) {
            this.tops[mode] = res.info.leaderboard
            if (res.info.self && res.info.self.score > 0) {
              let selfOnBoard = res.info.leaderboard.filter(player => player.name === res.info.self.name && player.score === res.info.self.score)
              if (selfOnBoard.length) {
                res.info.self.rank = res.info.leaderboard.indexOf(selfOnBoard[0]) + 1
              }
            }
            this.self[mode] = res.info.self
          }
        })
      })
    }
  }
</script>
<style lang="less" rel="stylesheet/less">
  @import "../../less/common";
  .voez-leaderboard {
    padding-bottom: env(safe-area-inset-bottom);

    -webkit-transform-style: preserve-3d;
    .vplus-leaderboard-self {
      position: absolute;
      left: 0;
      top: 0;
      width: 40%;
      max-width: 375px * 35%;
      height: 100%;
      background-color: #fff;
      box-shadow: 0 0 20px 40px #fff;
      box-sizing: border-box;
      padding: 10px 0 10px 15px;
      overflow-x: visible;
      white-space: nowrap;
      color: #6869a1;
    }
    .vplus-leaderboard-self-notes {
      position: relative;
      float: right;
      @squareSize: 44px;
      font-size: @squareSize / 2;
      margin-right: -@squareSize / 4  * 3;
      transform: scale(.8);
      transform-origin: 100% 0;

      width: @squareSize * 7;
      height: @squareSize * 4;
      .vplus-leaderboard-self-grade {
        position: absolute;
        z-index: 0;
        width: @squareSize * 2;
        height: @squareSize * 2;
        line-height: @squareSize * 2;
        text-align: center;

        top: @squareSize;
        right: @squareSize + 2px;

        font-size: @squareSize;
        color: #ffffff;

        &.fc {
          line-height: @squareSize * 2 - 12px;
          text-shadow: 0 0 10px #34B9F8;
        }
        &.ap {
          text-shadow: 0 0 10px #F8B427;
        }
        &.fc::after {
          content: 'Full Combo';
          display: block;
          line-height: 1;
          font-size: 12px;
          position: relative;
          top: -@squareSize / 2 + 3px;
        }
        &.ap::after {
          content: 'All Perfect';
        }
        .vplus-leaderboard-self-grade-bg {
          position: absolute;
          display: block;
          width: @squareSize * 2 * sqrt(2);
          height: @squareSize * 2 * sqrt(2);
          top: 50%;
          left: 50%;
          background-color: rgba(0, 0, 0, 0.4);
          border: 1px solid #ffffff;
          transform: translate3d(-50%, -50%, 0) rotate(45deg);
          box-sizing: border-box;
          overflow: hidden;
          z-index: -1;

          &::before, &::after {
            content: '';
            position: absolute;
            display: none;
            width: @squareSize * 2 * sqrt(2);
            height: @squareSize * 2 * sqrt(2);
            top: 50%;
            left: 50%;
            opacity: .4;
            box-sizing: border-box;
            z-index: -1;
          }
          &::before {
            transform: translate3d(-77%, -77%, 0);
          }
          &::after {
            transform: translate3d(-23%, -23%, 0);
          }
        }
        &.fc > .vplus-leaderboard-self-grade-bg {
          &::before, &::after {
            display: block;
            border: 2px solid lighten(#34B9F8, 20%);
          }
        }
        &.ap > .vplus-leaderboard-self-grade-bg {
          &::before, &::after {
            border-color: lighten(#F8B427, 20%);
          }
        }
      }
      .vplus-leaderboard-self-note-square {
        position: absolute;
        z-index: 0;
        width: @squareSize;
        height: @squareSize;
        line-height: @squareSize;
        text-align: center;
        &.mp {
          top: @squareSize / 2 + 1px;
          left: @squareSize / 2 + 1px;
          color: #FBD741;
        }
        &.perfect {
          top: @squareSize / 2 + 1px;
          left: @squareSize / 2 * 5 - 1px;
          color: #F1A873;

        }
        &.combo {
          top: @squareSize / 2 * 3;
          left: @squareSize / 2 * 3;
          color: #DD66CF;
        }
        &.ok {
          bottom: @squareSize / 2+ 1px;
          left: @squareSize / 2+ 1px;
          color: #50BDEE;

        }
        &.miss {
          bottom: @squareSize / 2+ 1px;
          left: @squareSize / 2 * 5 - 1px;
          color: #E8555A;
        }
        &::before {
          content: '';
          position: absolute;
          display: block;
          width: @squareSize * sqrt(2);
          height: @squareSize* sqrt(2);
          top: 50%;
          left: 50%;
          background-color: rgba(0, 0, 0, 0.4);
          border: 1px solid #ffffff;
          transform: translate3d(-50%, -50%, 0) rotate(45deg);
          box-sizing: border-box;
          z-index: -1;
        }
        &::after {
          display: block;
          line-height: 1;
          font-size: 12px;
          position: relative;
          top: -@squareSize / 4;
        }
        &.mp::after {
          content: 'Max';
        }
        &.perfect::after {
          content: 'Perfect';
        }
        &.combo::after {
          content: 'Combo';
        }
        &.ok::after {
          content: 'Ok';
        }
        &.miss::after {
          content: 'Miss';
        }
      }
    }

    .fade-transition {
      animation-duration: 400ms;
      animation-fill-mode: both;
    }
    .fade-enter {
      animation-name: fadeIn;
    }
    .fade-leave {
      animation-name: fadeOut;
    }
    @keyframes fadeIn {
      from {
        opacity: 0;
      }
      to {
        opacity: 1;
      }
    }
    @keyframes fadeOut {
      from {
        opacity: 1;
      }
      to {
        opacity: 0;
      }
    }
    .weui_cell {
      -webkit-transform: translate3d(0, 0, 0);
      -webkit-backface-visibility: hidden;
    }
    .voez-leaderboard-song-cover {
      width: 100%;
      height: 0;
      padding-top: 56.25%;

      position: relative;
      z-index: 0;
      &::after {
        content: '';
        display: block;
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        z-index: -1;
        background: url('http://voez.sevenoutman.com/voez/cover/default/750') center;
        background-size: cover;
      }
      & > * {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        display: block;
        overflow: hidden;
        pointer-events: none;
        z-index: 0;
        &.vplus-leaderboard-song-cover-box {
          z-index: 1;
        }
      }
    }
    .vux-sticky {
      z-index: 100;
      .vux-tab-item {
        .font-voez;
      }
    }
    .vux-tab-ink-bar-transition-forward {
      transition+_: right 0.3s cubic-bezier(0.35, 0, 0.25, 1), left 0.3s cubic-bezier(0.35, 0, 0.25, 1) 0.09s, background-color 300ms cubic-bezier(0.35, 0, 0.25, 1);
    }
    .vux-tab-ink-bar-transition-backward {
      transition+_: right 0.3s cubic-bezier(0.35, 0, 0.25, 1) 0.09s, left 0.3s cubic-bezier(0.35, 0, 0.25, 1), background-color 300ms cubic-bezier(0.35, 0, 0.25, 1);
    }
    .voez-leaderboard-song-info {
      display: none;
    }
  }

  .ipad .voez-leaderboard {
    padding: 30px;
    box-sizing: border-box;
    position: relative;
    .voez-leaderboard-song-cover {
      width: 50%;
      padding-top: 28.125%;
      border-radius: 5px;
      overflow: hidden;
      box-shadow: 0 1px 10px 0 rgba(0, 0, 0, .6);
      & + div {
        margin-top: 15px;
      }
    }

    .voez-leaderboard-song-info {
      position: absolute;
      top: 30px;
      right: 0;
      width: 50%;
      padding: 0 30px 0 15px;
      box-sizing: border-box;
      display: block;
    }
    .weui_cells_title {
      display: none;
    }
    .weui_cells::before, .weui_cells::after {
      display: none;
    }
  }
</style>
