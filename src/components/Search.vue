<template>
  <div class="search">
    <div class="form-group"><br>
      <button style="width:160px" :class="[showFullSearch ? 'btn-dark' : 'btn-light', 'btn']" @click="changeSearchToFull">Full Search</button>
      <button style="width:160px" :class="[showFullSearch ? 'btn-light' : 'btn-dark', 'btn']" @click="changeSearchToByName">Quick Search By Name</button>
    </div>
    <div v-show="!showFullSearch">
      <label>Space Name</label><br>
      <input size="22" type="text" v-model="name" placeholder="Enter name..." @keyup.enter="searchByName"/><br><br>
      <button :id="[showFullSearch ? '' : 'submitButton']" class="btn btn-primary submitButton" @click="searchByName">Search Spaces</button>
    </div>
    <div v-show="showFullSearch">
      <div>
        <h1>Meeting Space Criteria</h1>
        <p>Leave a criteria blank to indicate no preference.</p>
      </div>
      <div class="leftSearch">
        <div class="form-group">
          <label>Capacity</label>
          <NumberSlider v-on:change="capacityChanged" :allowAny="true" :capacity_prop="capacity"></NumberSlider>
        </div>
        <div class="form-group">
          <label>Meeting Day</label>
          <DatePicker v-on:change="startDateChanged" :date_prop="startDate"></DatePicker>
        </div>
        <div class="form-group">
          <label>Meeting Time</label>
          <br>
          <div class="timePicker">
            <TimePicker v-on:change="startTimeChanged" :time_prop="startTime"></TimePicker>
          </div>
          <div class="to">
            <p>to</p>
          </div>
          <div class="timePicker">
            <TimePicker v-on:change="endTimeChanged" :time_prop="endTime"></TimePicker>
          </div>
          <div class="clearFix"></div>
          <div class="text-danger">{{timeError}}</div>
        </div>
      </div>
      <div class="rightSearch">
        <div class="form-group themes">
          <label>Theme</label><br>
          <ul class="checkbox-grid">
            <li v-for="theme in $possibleThemes" :key="theme">
              <input type="checkbox" name="themeCheckbox" :value="theme" :id="theme" v-model="selectedThemes"/>
              <label :for="theme" class="checkboxLabel">{{theme.charAt(0).toUpperCase() + theme.slice(1)}}</label>
            </li>
          </ul>
        </div>
        <div class="form-group typeAhead">
          <TypeAhead :selectedAttributes="selectedAttributes" :allowCustom="false"></TypeAhead>
        </div>
      </div>
      <button id="submitButton" class="btn btn-primary submitButton" @click="fullSearch">Search Spaces</button>
    </div>
    <div class="clearFix"></div>
    <br>

    <br>
    <div v-if="searchCompleted">
      <h2 v-if="availableOnlyFilter.length === 1">Search Results: 1 space found</h2>
      <h2 v-else>Search Results: {{availableOnlyFilter.length}} spaces found</h2>
      <div class="form-group">
        <toggle-button v-if="!timeOmitted" @change="changeUserFilterKey" :value="false" :width=180 :height=25 :color="{checked: '#000', unchecked: '#444'}" :labels="{checked: 'Show all rooms', unchecked: 'Show only available rooms'}"/>
      </div>
      <div v-for="match in availableOnlyFilter" :key="match.email" :class="[{ 'searchLocationManyMissing': resultsLength == 'long' }, { 'searchLocationMedMissing': resultsLength == 'medium' }]" class="searchLocation col-lg-4 col-md-4 col-sm-6 col-xs-12">
        <div :class="{'matchTitle': match.matchPercent}">
          <p v-if="match.matchPercent" class="matchPercent" :class="[{ 'highMatch': match.matchPercent >= 80 }, { 'mediumMatch': match.matchPercent < 80 &&  match.matchPercent >= 50}, { 'lowMatch': match.matchPercent < 50 }]">{{match.matchPercent}}%</p>
          <h2>{{match.name}}</h2>
        </div>
        <div class="clearFix"></div>
        <p class="block-with-text">{{match.description}}</p>
        <router-link :to="'/space/' + match.email">
          <img :src="match.image" :alt="match.name + ' image'" class="img-fluid img-thumbnail searchImg">
        </router-link>
        <div v-if="!searchingByName && match.matchPercent !== 100">
          <div v-if="searchCriteria.capacity !== 0 && parseInt(searchCriteria.capacity) > parseInt(match.capacity)" class="missingCapacity">
            <ul>
              <li class="missingCapacity"><strong>Capacity NOT a match:</strong> has a capacity of {{match.capacity}}</li>
            </ul>
          </div>
          <div v-else>
            <p><strong>Capacity sufficient:</strong> has a capacity of {{match.capacity}}</p>
          </div>
          <div v-if="match.missThemes.length" class="missingThemes">
            <p><strong>Missing Themes</strong></p>
            <ul :class="{ 'missingItems': match.missThemes.length > 5 }">
              <li v-for="theme in match.missThemes" :key="theme">{{theme}}</li>
            </ul>
          </div>
          <div v-if="match.missAttributes.length" class="missingAttributes">
            <p><strong>Missing attributes</strong></p>
            <ul :class="{ 'missingItems': match.missAttributes.length > 5 }">
              <li v-for="attribute in match.missAttributes" :key="attribute">{{attribute}}</li>
            </ul>
          </div>
        </div>
        <div class="clearFix"></div>
        <div class="searchBtns">
          <router-link :to="'/space/' + match.email" class="btn btn-primary btnMargin">Details</router-link>
          <router-link :to="'/schedule-space/' + match.email + '/' + match.name" class="btn btn-primary btnMargin">Book</router-link>
          <router-link :to="'/edit-space/' + match.email" class="btn btn-primary btnMargin">Edit</router-link>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import NumberSlider from './NumberSlider'
import DatePicker from './DatePicker'
import TypeAhead from './TypeAhead'
import TimePicker from './TimePicker'
import axios from 'axios'

export default {
  name: 'Search',
  components: {
    'NumberSlider': NumberSlider,
    'DatePicker': DatePicker,
    'TypeAhead': TypeAhead,
    'TimePicker': TimePicker
  },
  computed: {
    availableOnlyFilter () {
      return this[this.userFilterKey]
    },
    all () {
      if (this.matches.length > 0) {
        return this.matches.slice().sort(this.sortByPercentage)
      }
      return this.matches
    },
    // filter out rooms that are labeled as busy
    available () {
      return this.matches.filter(function (el) {
        return !el.busy
      }).sort(this.sortByPercentage)
    }
  },
  data () {
    return {
      userFilterKey: 'available',
      searchCompleted: false,
      pageTitle: 'Search Meeting Spaces',
      matches: [],
      searchCriteria: {},
      numCriteria: 0,
      selectedThemes: [],
      selectedAttributes: [],
      resultsLength: '',
      startDate: new Date(), // default to today
      startTime: new Date(''), // empty date object
      endTime: new Date(''),
      capacity: 0,
      timeOmitted: false,
      showFullSearch: true,
      name: '', // name used for search by name
      searchingByName: false,
      timeError: ''
    }
  },
  mounted () {
    document.title = 'Search Spaces'

    var searchResults = JSON.parse(localStorage.getItem('searchResults'))
    if (searchResults) {
      this.matches = searchResults
      this.resultsLength = JSON.parse(localStorage.getItem('resultsLength'))
      this.searchCompleted = true
    }

    var searchCriteria = JSON.parse(localStorage.getItem('searchCriteria'))
    if (searchCriteria) {
      if (searchCriteria.startDate) {
        this.startDate = new Date(searchCriteria.startDate)
      }
      if (searchCriteria.startTime) {
        this.startTime = new Date(searchCriteria.startTime)
      }
      if (searchCriteria.endTime) {
        this.endTime = new Date(searchCriteria.endTime)
      }
      this.capacity = searchCriteria.capacity
      this.selectedThemes = searchCriteria.themes ? searchCriteria.themes : []
      this.selectedAttributes = searchCriteria.attributes ? searchCriteria.attributes : []
    }
  },
  methods: {
    sortByPercentage (a, b) {
      if (a.matchPercent < b.matchPercent) {
        return 1
      }
      if (a.matchPercent > b.matchPercent) {
        return -1
      }
      return 0
    },
    /** Is called when meeting day changes
    */
    startDateChanged (date) {
      this.startDate = date
    },
    /** Is called when start time changes
    */
    startTimeChanged (time) {
      this.startTime = time
    },
    /** Is called when end time changes
    */
    endTimeChanged (time) {
      this.endTime = time
    },
    /** Is called when capacity slider changes
    */
    capacityChanged (capacity) {
      this.capacity = capacity
    },
    /**
    * Returns a string of space delimited items from the given list of strings.
    * Returns empty string if list is empty.
    */
    getSpaceDelimitedItems (items) {
      var spaceDelimited = ''
      for (var j = 0; j < items.length; j++) {
        spaceDelimited += items[j]
        if (j < items.length - 1) {
          spaceDelimited += ' '
        }
      }
      return spaceDelimited
    },
    /**
    * Returns a string representation of the given date in the form YYYY-MM-DD
    */
    getFormattedDate (date) {
      var month = date.getMonth() + 1
      return date.getFullYear() + '-' + month + '-' + date.getDate()
    },
    /**
    * Returns a specifically formatted string representation of the give time
    */
    getFormattedTime (time) {
      var timezoneHours = -1 * (time.getTimezoneOffset() / 60)
      var timezoneString = '-0'
      if (timezoneHours > 0) {
        timezoneString = '+0'
      }
      timezoneString += Math.abs(timezoneHours)
      timezoneString += ':00'
      if (time != null) {
        return this.pad(time.getHours(), 2) + ':' + this.pad(time.getMinutes(), 2) + ':00' + timezoneString
      }
    },
    pad (n, width, z) {
      z = z || '0'
      n = n + ''
      return n.length >= width ? n : new Array(width - n.length + 1).join(z) + n
    },
    /**
    * Search by space name only
    */
    searchByName () {
      this.searchCompleted = false
      this.timeError = ''
      this.searchingByName = true
      this.timeOmitted = true
      this.$store.clearDates()
      this.$store.removeSearchCriteria()

      //  Edit distance signifies fuzziness. (If you're not sure what 'edit distance' means, look it up on the internet).
      //  While "magical", edit distance should be used with care.
      //  When user's search term consists of two or less letters
      //  then the edit distance of two will match any record in the dataset.
      //  Empirically, a "good" practice is to use edit distance of two for search terms of length
      //  five or more. It is also "reasonable" to use edit distance of one for search terms
      //  of length three or four. Edit distance should be 0 for search terms of length two or less.
      var editDistance = 0
      if (this.name.length >= 5) {
        editDistance = 2
      } else if (this.name.length >= 3) {
        editDistance = 1
      }

      var search = {
        'query': {
          'simple_query_string': {
            'fields': ['space.name'],
            //  ~N after a word signifies edit distance (fuzziness)
            //  See: https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html
            'query': this.name + '~' + editDistance
          }
        }
      }
      this.runSearch(search)
    },
    /**
    * Run a full search
    */
    fullSearch () {
      this.timeOmitted = isNaN(this.startTime.getTime()) || isNaN(this.endTime.getTime())
      if (!this.timeOmitted) {
        if (this.startTime >= this.endTime) {
          this.searchCompleted = false
          this.timeError = 'The start time must come before the end time'
          return
        } else {
          this.$store.setDates(this.startDate, this.startTime, this.endTime)
        }
      } else {
        this.startTime = new Date('')
        this.endTime = new Date('')
        this.$store.clearDates()
      }

      this.searchCompleted = false
      this.timeError = ''
      this.searchingByName = false
      this.userFilterKey = 'available'
      this.numCriteria = 0
      if (this.capacity !== 0) {
        this.numCriteria++
      }
      // search for half the capacity, but report capacity is not a match if actual capacity < desired capacity
      var searchCapacity = this.capacity / 2
      var searchObject = {}

      //  If neither theme nor any attributes is selected, then just search by capacity.
      if (this.selectedThemes.length === 0 && this.selectedAttributes.length === 0) {
        searchObject = {
          'query': {
            'range': {
              'space.capacity': {
                'gte': searchCapacity
              }
            }
          }
        }
        this.searchCriteria = {capacity: this.capacity}
      } else {
        var multisearch = this.getSpaceDelimitedItems(this.selectedThemes) + ' ' + this.getSpaceDelimitedItems(this.selectedAttributes)
        searchObject = {
          'query': {
            'bool': {
              'must': {
                'multi_match': {
                  'query': multisearch,
                  'fields': ['space.themes', 'space.attributes']
                }
              },
              'filter': {
                'range': {
                  'space.capacity': {
                    'gte': searchCapacity
                  }
                }
              }
            }
          }
        }
        this.searchCriteria = {capacity: this.capacity, themes: this.selectedThemes, attributes: this.selectedAttributes}
        this.numCriteria += this.selectedThemes.length + this.selectedAttributes.length
      }

      if (!this.timeOmitted) {
        this.searchCriteria.startDate = this.startDate
        this.searchCriteria.startTime = this.startTime
        this.searchCriteria.endTime = this.endTime
      }

      this.runSearch(searchObject)
    },
    /**
    * Run the search query
    */
    runSearch (searchObject) {
      var jsonStr = JSON.stringify(searchObject)
      var searchSize = '20'
      var searchUrl = this.$searchUrl + '/_search?from=0&size=' + searchSize

      this.$http.post(searchUrl, jsonStr, {
        headers: this.$defaultHeaders
      }).then(result => {
        this.matches = []
        let searchResults = result.body.hits.hits
        let lst = []
        Object.assign(lst, ...Object.values(searchResults).map(k => lst.push(k._id)))
        let emails = lst.filter(Boolean)

        if (emails.length > 0) {
          // if no time was provided, process the results without an availableList
          if (this.timeOmitted) {
            this.process(searchResults, false)
            return
          }
          let availabilityData = {
            'calendars': emails,
            'start_time': this.getFormattedDate(this.startDate) + 'T' + this.getFormattedTime(this.startTime),
            'end_time': this.getFormattedDate(this.startDate) + 'T' + this.getFormattedTime(this.endTime)
          }

          axios.put(this.$availabilityUrl, JSON.stringify(availabilityData), {
            headers: this.$defaultHeaders
          }).then(result => {
            let availableList = Object.values(result.data).map(el => {
              let filteredKeys = Object.keys(el).filter(key => {
                // only return available if it's not busy and there are no errors (indicating a private or non-existing email)
                return el[key].busy && el[key].busy.length === 0 && !(el[key].errors && el[key].errors.length > 0)
              })
              return filteredKeys[0]
            })
            this.process(searchResults, availableList)
          }, error => {
            console.error(error)
            alert('Looking up space availability failed.')
          })
        } else {
          this.matches = []
          this.finishedSearch()
        }
      }, error => {
        console.error(error)
        alert('Search failed.')
      })
    },
    /**
    * Process the search results and push results to this.matches
    */
    process (searchResults, availableList) {
      for (var n = 0; n < searchResults.length; n++) {
        var spaceId = searchResults[n]._id
        var entry = searchResults[n]._source.space

        if (!this.searchingByName) {
          // find matches
          var numMatches = 0
          if (this.searchCriteria.capacity !== 0 && parseInt(entry.capacity) >= parseInt(this.searchCriteria.capacity)) {
            numMatches = 1
          }
          var searchThemes = this.searchCriteria.themes
          var missingThemes = []
          if (searchThemes !== null && searchThemes !== undefined) {
            for (var i = 0; i < searchThemes.length; i++) {
              if (entry.themes.includes(searchThemes[i])) {
                numMatches++
              } else {
                missingThemes.push(searchThemes[i])
              }
            }
          }

          var searchAttributes = this.searchCriteria.attributes
          var missingAttributes = []
          if (searchAttributes !== null && searchAttributes !== undefined) {
            for (var j = 0; j < searchAttributes.length; j++) {
              if (entry.attributes.includes(searchAttributes[j])) {
                numMatches++
              } else {
                missingAttributes.push(searchAttributes[j])
              }
            }
          }
          if (!this.resultsLength === 'long' && (missingThemes.length >= 2 || missingAttributes.length >= 2)) {
            this.resultsLength = 'medium'
            if (missingThemes.length >= 4 || missingAttributes.length >= 4) {
              this.resultsLength = 'long'
            }
          }
          // if there is no availableList, default to not busy
          let isBusy = availableList ? !availableList.includes(spaceId) : false
          // console.log(entry.name + ' is ' + (isBusy ? 'busy' : 'available'))
          this.matches.push({
            name: entry.name,
            description: entry.description,
            image: entry.image,
            theme: entry.themes,
            attributes: entry.attributes,
            email: spaceId,
            busy: isBusy,
            capacity: entry.capacity,
            missThemes: missingThemes,
            missAttributes: missingAttributes,
            //  Match percent: Make sure that match percent is never greater than 100%.
            //  This could happen when no criteria are selected and therefore numCriteria is zero.
            matchPercent: Math.min(100, Math.round((numMatches / this.numCriteria) * 100))
          })
        } else {
          // if this was a search by name, only this information is needed
          this.matches.push({
            name: entry.name,
            description: entry.description,
            image: entry.image,
            email: spaceId
          })
        }
      }
      this.finishedSearch()
    },
    /**
    * Stores the search criteria in localStorage, so that it is
    * available if a user clicks on a space and then navigates
    * back to the search page. Also scrolls to the results.
    */
    finishedSearch () {
      this.searchCompleted = true
      localStorage.setItem('searchCriteria', JSON.stringify(this.searchCriteria))
      localStorage.setItem('resultsLength', JSON.stringify(this.resultsLength))
      localStorage.setItem('searchResults', JSON.stringify(this.matches))
      // scroll to search results
      this.$nextTick(function () {
        // scroll down to the results
        document.getElementById('submitButton').scrollIntoView({block: 'start', inline: 'nearest', behavior: 'smooth'})
      })
    },
    /**
    * Switch filter key between 'all' and 'available'
    */
    changeUserFilterKey (sliderState) {
      if (sliderState.value) {
        this.userFilterKey = 'all'
      } else {
        this.userFilterKey = 'available'
      }
    },
    /**
    * Hide any previous search results and switch search type to full search
    */
    changeSearchToFull () {
      this.searchCompleted = false
      this.showFullSearch = true
    },
    /**
    * Hide any previous search results and switch search type to search by name
    */
    changeSearchToByName () {
      this.searchCompleted = false
      this.showFullSearch = false
    }
  }
}
</script>

<style scoped>
</style>
