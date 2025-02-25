<script>

import _ from 'lodash'
import api from 'core/api'
import utils from 'core/utils'
import { defaultCodeLanguage, findNextLevelsBySession, getLevelStatusMap } from 'ozaria/site/common/ozariaUtils'
import { mapActions, mapGetters, mapMutations } from 'vuex'
import LayoutChrome from '../../common/LayoutChrome'
import LayoutCenterContent from '../../common/LayoutCenterContent'
import LayoutAspectRatioContainer from 'ozaria/site/components/common/LayoutAspectRatioContainer'
import UnitMapBackground from './common/UnitMapBackground'
import HoC2019Modal from './hoc2019modal/index'
import ClassroomLib from '../../../../../app/models/ClassroomLib'
import Classroom from 'models/Classroom'

export default Vue.extend({
    components: {
      'layout-chrome': LayoutChrome,
      'layout-center-content': LayoutCenterContent,
      'layout-aspect-ratio-container': LayoutAspectRatioContainer,
      'unit-map-background': UnitMapBackground,
      'hoc-2019-modal': HoC2019Modal
    },

    props: {
      campaign: { // campaign slug / campaign id
        type: String,
        required: true
      },

      campaignPage: {
        type: String,
        default: undefined
      },

      courseInstanceId: {
        type: String,
        default: undefined
      },

      courseId: {
        type: String,
        default: undefined
      },

      codeLanguage: { // used for non-classroom users
        type: String,
        default: undefined
      }
    },

    data: () => ({
      campaignData: {},
      levels: {},
      classroomLevelMap: {},
      classroom: {},
      levelSessions: [],
      levelStatusMap: {},
      dataLoaded: false,
      nextLevelOriginal: '',
      ambientSound: undefined,
      hocCourseInstanceId: undefined,
      openHoC2019Modal: true,
    }),

    computed: {
      ...mapGetters({
        getCampaignData: 'campaigns/getCampaignData',
        currentLevelsList: 'unitMap/getCurrentLevelsList'
      }),

      computedCodeLanguage: function () {
        if (me.isStudent() && this.classroom) {
          return (this.classroom.aceConfig || {}).language
        }
        return this.codeLanguage || utils.getQueryVariable('codeLanguage') || defaultCodeLanguage
      },

      computedCourseInstanceId: function () {
        if (this.hocActivity && !me.isAnonymous()) {
          return this.courseInstanceId || utils.getQueryVariable('course-instance') || this.hocCourseInstanceId
        }
        return this.courseInstanceId || utils.getQueryVariable('course-instance')
      },

      computedCourseId: function () {
        if (this.hocActivity && !me.isAnonymous()) {
          return this.courseId || utils.getQueryVariable('course') || this.hocCourseId
        }
        return this.courseId || utils.getQueryVariable('course')
      },

      hocCourseId: function () {
        return utils.hourOfCodeOptions.courseId
      },

      computedCampaignPage: function () {
        return parseInt(this.campaignPage || utils.getQueryVariable('campaign-page')) || (this.levels[this.nextLevelOriginal] || {}).campaignPage || 1
      },

      hocActivity: function () {
        return utils.hourOfCodeOptions.campaignId === this.campaignData._id
      },

      showHoC2019Modal () {
        return utils.getQueryVariable('hour_of_code') && this.hocActivity && me.isAnonymous() && this.openHoC2019Modal
      },

      redirectUrl: function () {
        if (me.isTeacher() && !this.computedCourseId) {
          return '/teachers'
        } else if (me.isStudent() && !this.computedCourseInstanceId && !this.hocActivity) {
          return '/students'
        }
        // For students playing hoc activity(ch1), its possible that they dont have a course-instance
        // (example, coco students with no oz class/who sign up without class code using hoc save progress modal)
        // They should be allowed access to unit map, and their ch1 progress from hoc will be shown
        return null
      },
      title: function () {
        return utils.i18n(this.campaignData, 'name')
      }
    },

    watch: {
      campaign: async function () {
        await this.loadCampaign()
        await this.buildUnitMapData()
      }
    },

    async mounted () {
      await this.loadCampaign()

      // Fetch the hoc course instance for students playing hoc activity
      if (me.isStudent() && this.hocActivity && !this.computedCourseInstanceId) {
        this.hocCourseInstanceId = await me.getHocCourseInstanceId()
      }

      if (this.redirectUrl) {
        return application.router.navigate(this.redirectUrl, { trigger: true })
      }

      await this.buildUnitMapData()
      this.playAmbientSound()
    },

    beforeDestroy () {
      if (this.ambientSound) {
        this.fadeAndStopTrack({
          track: 'background',
          to: 0,
          duration: 1500,
          unload: true
        })
      }
    },

    methods: {
      ...mapActions({
        fetchCampaign: 'campaigns/fetch',
        buildLevelsData: 'unitMap/buildLevelsData',

        playSound: 'audio/playSound',
        fadeTrack: 'audio/fadeTrack',
        fadeAndStopTrack: 'audio/fadeAndStopTrack',
        stopTrack: 'audio/stopTrack'
      }),

      ...mapMutations({
        setUnitMapUrlDetails: 'layoutChrome/setUnitMapUrlDetails'
      }),

      playAmbientSound () {
        const ambientSoundConfig = (this.campaignData || {}).ambientSound || {}
        const soundFiles = Object.values(ambientSoundConfig)

        if (soundFiles.length === 0 || this.ambientSound) {
          return
        }

        this.ambientSound = this.playSound({
          track: 'background',
          src: soundFiles.map(f => `/file/${f}`),
          loop: true,
          volume: 0.1
        })

        this.fadeTrack({
          track: 'background',
          from: 0.1,
          to: 1,
          duration: 1000
        })
      },

      // When hoc modal is closed without signing up, this is registered so that progress modal is shown after 25 min
      registerHocProgressModalCheck () {
        if ((utils.hourOfCodeOptions || {}).progressModalAfter && me.isAnonymous() && !window.sessionStorage.getItem('hoc_progress_modal_time')) {
          window.sessionStorage.setItem('hoc_progress_modal_time', new Date().getTime() + utils.hourOfCodeOptions.progressModalAfter)
          utils.registerHocProgressModalCheck()
        }
        // Note: Once the interval is registered and if hoc modal is opened again, or page is refreshed
        // then, this interval will still keep running -> It might result in some rare edge cases
      },

      closeHocModal () {
        this.openHoC2019Modal = false
        this.registerHocProgressModalCheck()
        _.defer(() => {
          $(this.$el).find('img.level-dot-image.next').parent().focus()
        })
      },

      async loadCampaign () {
        await this.fetchCampaign({
          campaignHandle: this.campaign,
          courseInstanceId: this.computedCourseInstanceId,
          courseId: this.computedCourseId
        })
        this.campaignData = this.getCampaignData({
          campaignHandle: this.campaign,
          courseInstanceId: this.computedCourseInstanceId,
          courseId: this.computedCourseId
        })

        if (!me.hasCampaignAccess(this.campaignData)) {
          alert('You must obtain a student license to access this page.')
          return application.router.navigate('/', { trigger: true })
        }
      },

      async buildUnitMapData () {
        try {
          this.dataLoaded = false
          // Set current campaign id and unit map URL details for acodus chrome
          this.setUnitMapUrlDetails({ courseId: this.computedCourseId, courseInstanceId: this.computedCourseInstanceId })

          if (this.computedCourseInstanceId) {
            await this.buildClassroomLevelMap()
          }
          await this.buildLevelsData({ campaignHandle: this.campaign, courseInstanceId: this.computedCourseInstanceId, courseId: this.computedCourseId })
          this.levels = this.currentLevelsList
          if (!me.isTeacher()) {
            this.levelSessions = await api.users.getLevelSessions({ userID: me.get('_id') })
            this.createLevelStatusMap()
            this.determineNextLevel()
          }
          this.dataLoaded = true
        } catch (err) {
          console.error('ERROR:', err)
          // TODO: update after a consistent error handling strategy is decided
          noty({ text: 'Error in creating unit map data', layout: 'topCenter', type: 'error' })
        }
      },

      async buildClassroomLevelMap () {
        const courseInstance = await api.courseInstances.get({ courseInstanceID: this.computedCourseInstanceId })
        const courseId = courseInstance.courseID
        if (this.computedCourseId && this.computedCourseId !== courseId) {
          // TODO handle_error_ozaria
          noty({ text: 'Invalid data', layout: 'topCenter', type: 'error' })
          console.error('Course instance does not contain course id ', this.computedCourseId)
        }
        const classroomId = courseInstance.classroomID

        const classroom = this.classroom = await api.classrooms.get({ classroomID: classroomId })
        const classroomCourseLevels = _.find(classroom.courses, { _id: courseId }).levels

        for (const level of classroomCourseLevels) {
          this.classroomLevelMap[level.original] = level
        }
      },

      createLevelStatusMap () {
        // Remove the level sessions for the levels played in another language - for the classroom version of unit map
        if (me.isStudent() && this.classroomLevelMap && this.classroom) {
          for (let session of this.levelSessions.slice()) {
            const classroomLevel = this.classroomLevelMap[session.level.original]
            if (!classroomLevel) { continue }
            const expectedLanguage = classroomLevel.primerLanguage || this.classroom.aceConfig.language
            if (session.codeLanguage !== expectedLanguage) {
              this.levelSessions.splice(this.levelSessions.indexOf(session), 1)
            }
          }
        } else { // for anon/individual/hoc students without ch1 class
          this.levelSessions = this.levelSessions.filter((s) => s.codeLanguage === this.computedCodeLanguage)
        }
        this.levelStatusMap = getLevelStatusMap(this.levelSessions)
      },

      determineNextLevel () { // set .next and .locked for this.levels
        if (this.computedCourseInstanceId || this.campaignData.type === 'course') {
          const classroom = new Classroom(this.classroom)
          this.nextLevelOriginal = findNextLevelsBySession(this.levelSessions, this.levels, this.levelStatusMap, classroom, this.computedCourseId)
          this.setCompletedLevels()
          this.setNextLevels()
          this.setUnlockedLevels()
        }
      },

      setUnlockedLevels () {
        let previousLocked = false
        let previousOptional = false

        for (let level in this.levels) {

          const lockedByTeacher = ClassroomLib.isModifierActiveForStudent(this.classroom, me.get('_id'), this.computedCourseId, this.levels[level].original, 'locked')
          const optional = ClassroomLib.isModifierActiveForStudent(this.classroom, me.get('_id'), this.computedCourseId, this.levels[level].original, 'optional')

          let lock = false

          if (this.levelStatusMap[level] || this.levels[level].first || this.nextLevelOriginal === level) {
            lock = lockedByTeacher || false
          } else {
            lock = lockedByTeacher || previousLocked || (!previousOptional) || false
          }

          if (this.levels[level].complete === true) {
            lock = false
          }

          if ((lock === true || this.levels[level].complete !== true) && !optional) {
            // Ensure all levels after the first non-optional assigned level will be locked.
            previousLocked = true
          }
          previousOptional = optional
          this.levels[level].optional = optional
          this.levels[level].locked = lock
        }
      },

      setNextLevels () {
        if (this.nextLevelOriginal && this.levels[this.nextLevelOriginal]) {
          this.levels[this.nextLevelOriginal].next = true
        }
      },

      setCompletedLevels () {
        for (let level in this.levelStatusMap) {
          if(!this.levels[level]){
            continue
          }
          if (this.levelStatusMap[level] === 'complete') {
            this.levels[level].complete = true
          } else if (this.levelStatusMap[level] === 'started') {
            this.levels[level].started = true
          }
        }
      }
    }
  })
</script>

<template>
  <layout-chrome
    :title="title"
  >
    <layout-center-content>
      <layout-aspect-ratio-container
        v-if="dataLoaded"
        id="unit-map-container"
        :aspect-ratio="1266 / 668"
      >
        <unit-map-background
          :campaign-data="campaignData"
          :levels="levels"
          :campaign-page="computedCampaignPage"
          :course-id="computedCourseId"
          :course-instance-id="computedCourseInstanceId"
          :code-language="computedCodeLanguage"
        />
      </layout-aspect-ratio-container>
    </layout-center-content>
    <hoc-2019-modal
      v-if="showHoC2019Modal"
      @closeModal="closeHocModal()"
    />
  </layout-chrome>
</template>

<style scoped>
#unit-map-container{
  width: 100%;
  height: 100%;
  max-width: 1266px;
  max-height: 668px;
}
</style>
