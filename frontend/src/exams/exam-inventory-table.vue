<template>
  <div>
    <b-form inline class="ml-3">
      <b-input-group>
        <b-input-group-prepend><label class="mx-1 pt-1 my-auto label-text">Search</label></b-input-group-prepend>
        <b-input size="sm" class="mb-1 mt-3" v-model="filter"></b-input>
      </b-input-group>
      <b-input-group class="ml-3">
        <b-input-group-prepend>
          <label class="mx-1 pt-1 my-auto label-text">Filters</label>
        </b-input-group-prepend>
        <b-btn-group horizontal class="pt-2">
          <b-btn value="all"
                 size="sm"
                 :pressed="expiryFilter==='all'"
                 @mousedown="storeLocally"
                 @click="expiryFilter='all'"><span class="mx-2">All</span></b-btn>
          <b-btn value="expired"
                 size="sm"
                 :pressed="expiryFilter==='expired'"
                 @mousedown="storeLocally"
                 @click="expiryFilter='expired'">Expired</b-btn>
          <b-btn value="current"
                 size="sm"
                 @mousedown="storeLocally"
                 :pressed="expiryFilter==='current'"
                 @click="expiryFilter='current'">Current</b-btn>
        </b-btn-group>
        <b-btn-group horizontal class="ml-2 pt-2">
          <b-btn value="both"
                 size="sm"
                 @mousedown="storeLocally"
                 :pressed="bookedFilter==='both'"
                 @click="bookedFilter='both'"><span class="mx-2">Both</span></b-btn>
          <b-btn value="unbooked"
                 size="sm"
                 @mousedown="storeLocally"
                 :pressed="bookedFilter==='unbooked'"
                 @click="bookedFilter='unbooked'">Un-Booked</b-btn>
          <b-btn value="booked"
                 size="sm"
                 @mousedown="storeLocally"
                 :pressed="bookedFilter==='booked'"
                 @click="bookedFilter='booked'">Booked</b-btn>
        </b-btn-group>
        <b-btn-group horizontal class="ml-2 pt-2">
          <b-btn value="both"
                 size="sm"
                 @mousedown="storeLocally"
                 :pressed="groupFilter==='both'"
                 @click="groupFilter='both'"><span class="mx-2">Both</span></b-btn>
          <b-btn value="unbooked"
                 size="sm"
                 @mousedown="storeLocally"
                 :pressed="groupFilter==='individual'"
                 @click="groupFilter='individual'">Individual</b-btn>
          <b-btn value="booked"
                 size="sm"
                 @mousedown="storeLocally"
                 :pressed="groupFilter==='group'"
                 @click="groupFilter='group'">Group</b-btn>
        </b-btn-group>
      </b-input-group>
    </b-form>
    <div :style="tableStyle" class="exam-table-holder my-0 mx-3">
      <b-table :items="filteredExams"
               :fields=getFields
               head-variant="light"
               style="border: 1px solid dimgrey"
               empty-text="There are no exams that match this filter criteria"
               small
               outlined
               @row-clicked="clickRow"
               hover
               show-empty
               :filter="filter">
        <template slot="student_name" slot-scope="row">
          {{ row.item === 'group' ? 'Group Exam' : row.item }}
        </template>
      <template slot="exam_received" slot-scope="row">
        {{ row.item.exam_received === 0 ? 'No' : 'Yes' }}
      </template>
        <template slot="booking_"
      <template slot="invigilator" slot-scope="row">
        {{ getInvigilator(row) }}
      </template>
      <template slot="expiry_date" slot-scope="row">
        {{ row.item.expiry_date.split('T')[0] }}
      </template>
      <template slot="location" slot-scope="row">
        <template v-if="!row.item.offsite_location">
          <span v-if="row.item.booking">
            <b-btn variant="link" class="open-cal-link" @click="openEditBooking">
                {{ row.item.booking.room.room_name }}
            </b-btn>
          </span>
          <span v-else><b-btn variant="link" class="schedule-link" to="/booking">Schedule</b-btn></span>
        </template>
        <span v-if="row.item.offsite_location">
          <b-btn variant="link" class="view-details-link" @click.stop="row.toggleDetails">
            {{ row.detailsShowing ? 'Hide' : 'Show'}} Details
          </b-btn>
        </span>
      </template>
        <template slot="row-details" slot-scope="row">
          <span class="mb-2 mt-0 ml-3"><b>Location: </b>{{ row.item.offsite_location }}</span>
        </template>
      <template slot="actions" slot-scope="row">
        <b-dropdown variant="link"
                    no-caret
                    size="sm"
                    class="pl-0 ml-0 mr-3"
                    id="nav-dropdown"
                    right>
          <template slot="button-content">
            <font-awesome-icon icon="caret-down"
                               style="padding: -2px; margin: -2px; font-size: 1rem; color: dimgray"/>
          </template>
          <b-dropdown-item size="sm" @click.stop="editInfo(row.item, row.index)">Edit Row</b-dropdown-item>
          <b-dropdown-item size="sm" @click.stop="returnExamInfo(row.item, row.index)">Return Exam</b-dropdown-item>
          <b-dropdown-item v-if=row.item.booking size="sm" @click="updateBookingRoute(row.item, row.index)">Update Booking</b-dropdown-item>
        </b-dropdown>
      </template>
    </b-table>
    </div>
    <EditExamModal v-if="showEditExamModalVisible"></EditExamModal>
    <ReturnExamModal v-if="showReturnExamModalVisible"></ReturnExamModal>
  </div>
</template>

<script>
  import { mapActions, mapGetters, mapMutations, mapState } from 'vuex'
  import EditExamModal from './edit-exam-form-modal'
  import ReturnExamModal from './return-exam-form-modal'
  import moment from 'moment'
  import SuccessExamAlert from './success-exam-alert'
  import FailureExamAlert from './failure-exam-alert'

  export default {
    name: "ExamInventoryTable",
    components: { EditExamModal, ReturnExamModal, SuccessExamAlert, FailureExamAlert },
    props: ['mode'],
    mounted() {
      this.getBookings().then(bookings => {
        this.events = bookings
      })
      this.getWidth()
      this.getExams()
      this.$nextTick(function() {
        window.addEventListener('resize', () => { this.getWidth() })
      })
    },
    data() {
      return {
        tableStyle: null,
        expiryFilter: 'all',
        bookedFilter: 'both',
        groupFilter: 'both',
        filter: null,
        events: null,
        fields: [
          {key: 'office.office_name', label: 'Office', sortable: true, thStyle: 'width: 8%'},
          {key: 'event_id', label: 'Event ID', sortable: true, thStyle: 'width: 6%' },
          {key: 'exam_name', label: 'Exam Name', sortable: true, thStyle: 'width: 11%' },
          {key: 'exam_method', label: 'Method', sortable: true, thStyle: 'width: 5%' },
          {key: 'expiry_date', label: 'Expiry Date', sortable: true, thStyle: 'width: 8%' },
          {key: 'exam_received', label: 'Received?', sortable: true, thStyle: 'width: 5%' },
          {key: 'examinee_name', label: 'Student Name', sortable: true, thStyle: 'width: 12%' },
          {key: 'notes', label: 'Notes', sortable: true, thStyle: 'width: 21%' },
          {key: 'invigilator', label: 'Invigilator', sortable: true, thStyle: 'width: 10%' },
          {key: 'location', label: 'Location', sortable: true, thStyle: 'width: 9%' },
          {key: 'actions', label: 'Actions', sortable: true, thStyle: 'width: 5%' }
        ],
        bookingRouteString: '',
      }
    },
    computed: {
      ...mapGetters(['calendar_events', 'exam_inventory', 'role_code',]),
      ...mapState([
        'bookings',
        'calendarSetup',
        'exams',
        'showEditExamModalVisible',
        'showExamInventoryModal',
        'showReturnExamModalVisible',
        'user',
      ]),
      filteredExams() {
        let exams = this.exam_inventory || []
        let filtered = []
        switch (this.expiryFilter) {
          case 'all':
            filtered = exams
            break
          case 'expired':
            filtered = exams.filter(ex=>moment(ex.expiry_date).isBefore(moment(), 'day'))
            break
          case 'current':
            filtered = exams.filter(ex=>moment(ex.expiry_date).isSameOrAfter(moment(), 'day'))
            break
          default:
            filtered = exams
            break
        }
        let moreFiltered = []
        switch (this.bookedFilter) {
          case 'both':
            moreFiltered = filtered
            break
          case 'unbooked':
            moreFiltered = filtered.filter(ex=>!ex.booking)
            break
          case 'booked':
            moreFiltered = filtered.filter(ex=>ex.booking)
            break
          default:
            moreFiltered = filtered
            break
        }
        let evenMoreFiltered = []
        switch (this.groupFilter) {
          case 'both':
            evenMoreFiltered = moreFiltered
            break
          case 'individual':
            evenMoreFiltered = moreFiltered.filter(ex=>!ex.offsite_location)
            break
          case 'group':
            evenMoreFiltered = moreFiltered.filter(ex=>ex.offsite_location)
            break
          default:
            evenMoreFiltered = moreFiltered
            break
        }
        return evenMoreFiltered
      },
      selectedExams() {
        if (this.showExamInventoryModal) {
          return this.exam_inventory
        }
        return this.exams
      },
      getFields() {
        if (this.role_code === "LIAISON") {
          return this.fields
        } else {
          let returnFields = this.fields
          let index = this.fields.findIndex(x => x.key === "office.office_name")
          returnFields.splice(index, 1)
          return returnFields
        }
      }

    },
    methods: {
      ...mapActions(['getExams', 'getBookings']),
      ...mapMutations([
        'navigationVisible',
        'setSelectedExam',
        'toggleCalendarControls',
        'toggleExamInventoryModal',
        'toggleScheduling',
        'toggleSchedulingIndicator',
        'toggleEditExamModalVisible',
        'setEditExamInfo',
        'toggleReturnExamModalVisible',
        'setReturnExamInfo'
      ]),
      storeLocally(e) {
        console.log(e)
      },
      handleExpiryFilter(e) {
        this.expiryFilter = e.target.value
      },
      openEditBooking() {

      },
      getWidth() {
        this.tableStyle = {width: `${window.innerWidth - 40}px`}
      },
      handleBookedFilter(e) {
        this.bookedFilter = e.target.value
      },
      resetButtons() {
        this.buttons.all = 'btn-secondary'
        this.buttons.current = 'btn-secondary'
        this.buttons.expired = 'btn-secondary'
      },
      getInvigilator(row) {
        if (this.events) {
          let bookingObj = this.events.find(event=>event.booking_id==row.item.booking_id)
          if (bookingObj && bookingObj.invigilator) {
            return bookingObj.invigilator.invigilator_name
          }
          return ''
        }
        return ''
      },
      clickRow(e) {
        if (this.showExamInventoryModal) {
          this.$root.$emit('options', {name: 'selectable', value: true})
          this.navigationVisible(false)
          this.setSelectedExam(e)
          this.toggleCalendarControls(false)
          this.toggleExamInventoryModal(false)
          this.toggleScheduling(true)
          this.toggleSchedulingIndicator(true)
        }
      },
      editInfo(item) {
        this.toggleEditExamModalVisible(true)
        this.setEditExamInfo(item)
      },
      returnExamInfo(item, index) {
        this.toggleReturnExamModalVisible(true)
        this.setReturnExamInfo(item)
      },
      updateBookingRoute(item) {
        let bookingRoute = '/booking/'
        let rowDate = moment(item.booking.start_time).format('YYYY-MM-DD')
        let dateConcat = bookingRoute.concat(rowDate)
        this.$router.push(dateConcat)
      }
    },
  }
</script>

<style scoped>
  .open-cal-link {
    text-decoration: #007bff underline !important;
    text-underline-position: under;
    font-size: .85rem;
    color: #007bff !important;
    font-weight: 300;
  }
  .view-details-link {
    text-decoration: #007bff underline !important;
    text-underline-position: under;
    font-size: .85rem;
    color: #007bff !important;
    font-weight: 300;
  }
  .view-details-link:hover {
    font-weight: 600;
  }
  .open-cal-link:hover {
    font-weight: 600;
  }
  .schedule-link {
    text-decoration: #28a745 underline !important;
    text-underline-position: under;
    font-size: .85rem;
    color: #28a745 !important;
    font-weight: 300;
  }
  .schedule-link:hover {
    font-weight: 600;
  }
  .btn {
    border: none !important;
  }
  .label-text {
    font-size: .9rem;
  }
  .btn {
    border: none !important;
    box-shadow: none !important;
    transition: none !important;
  }
  .btn:active, .btn.active {
    background-color: darkgrey !important;
  } color: blue !important;
  .exam-table-holder {
    border: 1px solid dimgrey;
  }
</style>
