<template lang="pug">
div
  h3.mb-0 {{ $t('activityFor') }} {{ periodReadable }}

  div.mb-2
    ul.list-group.list-group-horizontal-md.mb-3(style="font-size: 0.9em; opacity: 0.7")
      li.list-group-item.pl-0.pr-3.py-0(style="border: 0")
        | #[b {{ $t('host') }}] {{ host }}
      li.list-group-item.pl-0.pr-3.py-0(style="border: 0")
        | #[b {{ $t('timeActive') }}] {{ $store.state.activity.active.duration | friendlyduration }}

  div.mb-2.d-flex
    div
      b-input-group
        b-input-group-prepend
          b-button.px-2(:to="link_prefix + '/' + previousPeriod() + '/' + subview",
                   variant="outline-dark")
            icon(name="arrow-left")
        b-select.px-2(:value="periodLength", :options="['day', 'week', 'month']",
                 @change="(periodLength) => setDate(_date, periodLength)")
        b-input-group-append
          b-button.px-2(:to="link_prefix + '/' + nextPeriod() + '/' + subview",
                   :disabled="nextPeriod() > today", variant="outline-dark")
            icon(name="arrow-right")

    div.mx-2(v-if="periodLength === 'day'")
      input.form-control.px-2(id="date" type="date" :value="_date" :max="today"
                         @change="setDate($event.target.value, periodLength)")

    div.ml-auto
      b-button-group
        b-button.px-2(@click="refresh(true)", variant="outline-dark")
          icon(name="sync")
          span.d-none.d-md-inline
            |  {{ $t('refresh') }}

  aw-periodusage(:periodusage_arr="periodusage", @update="setDate")

  ul.row.nav.nav-tabs.mt-3.px-3
    li.nav-item(v-for="view in views")
      router-link.nav-link(:to="{ name: 'activity-view', params: {...$route.params, view_id: view.id}}" :class="{'router-link-exact-active': currentView.id == view.id}")
        h6 {{view.name}}

    div.edit(v-if="editing" style="margin: auto 1rem 1rem auto;").mt-2
      div.d-flex.flex-row-reverse
        b-button(variant="outline-dark" @click="discard(); editing = !editing;")
          icon(name="times")
          span {{ $t('cancel') }}
        b-button.mr-2(variant="success" @click="save(); editing = !editing;")
          icon(name="save")
          span {{ $t('save') }}
        b-button.mr-2(variant="warning" size="sm" @click="restoreDefaults();")
          icon(name="undo")
          span {{ $t('restore') }}
        b-button.mr-2(variant="danger" size="sm" @click="remove();")
          icon(name="trash")
          span {{ $t('remove') }}

    div(v-else style="margin-left:auto; margin-bottom: 1rem; margin-right: 15px;").d-flex.flex-row-reverse.mt-2
      li.nav-item
        a.nav-link(@click="$refs.new_view.show()")
          h6
            icon(name="plus")
            span.d-none.d-md-inline
              | {{ $t('newView') }}
      li.nav-item
        a.nav-link(@click="editing = !editing")
          h6
            icon(name="edit")
            span.d-none.d-md-inline
              | {{ $t('editView') }}
      a.ml-2.info(id="info")
        icon(name="info-circle")
      b-tooltip(target="info" :title="$t('editViewTooltip')")

  b-modal(id="new_view" ref="new_view" :title="$t('newView')" @show="resetModal" @hidden="resetModal" @ok="handleOk")
    div.my-1
      b-input-group.my-1(prepend="ID")
        b-form-input(v-model="new_view.id")
      b-input-group.my-1(prepend="Name")
        b-form-input(v-model="new_view.name")

  div
    router-view

  div
    hr
    h5 Options

    div.row
      div.col-md-6
        b-form-group(label="Show/filter category" label-cols="5" label-cols-lg="4")
          b-form-select(v-model="filterCategory", :options="categories")

    aw-devonly
      b-btn(id="load-demo", @click="load_demo")
        | Load demo data
</template>

<style lang="scss" scoped>
$buttoncolor: #ddd;
$bordercolor: #ddd;

.nav {
  border-bottom: 2px solid $bordercolor;

  .nav-item {
    margin-bottom: -2px;

    .nav-link {
      background-color: $buttoncolor;
      margin: 0 0.2em 0 0.2em;
      padding: 0.4em 0.5em 0.1em 0.5em;
      border: 2px solid $bordercolor;
      border-bottom: none;
      border-radius: 0.5rem 0.5rem 0 0;
      color: #111 !important;
      cursor: pointer;

      &:hover {
        background-color: #f8f8f8;
      }

      &.router-link-exact-active {
        background-color: #fff;
        color: #333 !important;

        &:hover {
          background-color: #fff;
        }
      }
    }
  }
}
.info {
  color: #343a40;
  margin-top: 7px;
  margin-right: 3px;
}
</style>

<script>
import moment from 'moment';
import { get_day_start_with_offset, get_today } from '~/util/time';
import _ from 'lodash';

import 'vue-awesome/icons/arrow-left';
import 'vue-awesome/icons/arrow-right';
import 'vue-awesome/icons/sync';
import 'vue-awesome/icons/plus';
import 'vue-awesome/icons/edit';
import 'vue-awesome/icons/times';
import 'vue-awesome/icons/save';
import 'vue-awesome/icons/info-circle';

export default {
  name: 'Activity',
  props: {
    host: String,
    date: {
      type: String,
      // NOTE: This does not work as you'd might expect since the default is set on
      // initialization, which would lead to the same date always being returned,
      // even if the day has changed.
      // Instead, use the computed _date.
      //default: get_today(),
    },
    periodLength: {
      type: String,
      default: 'day',
    },
  },
  data: function () {
    return {
      today: get_today(),
      filterCategory: null,
      new_view: {},
      editing: false,
    };
  },
  computed: {
    views: function () {
      return this.$store.state.views.views;
    },
    currentView: function () {
      return this.views.find(v => v.id == this.$route.params.view_id) || this.views[0];
    },
    _date: function () {
      return this.date || get_today();
    },
    subview: function () {
      return this.$route.meta.subview;
    },
    categories: function () {
      const cats = this.$store.getters['categories/all_categories'];
      const entries = cats.map(c => {
        return { text: c.join(' > '), value: c };
      });
      return [
        { text: 'All', value: null },
        { text: 'Uncategorized', value: ['Uncategorized'] },
      ].concat(entries);
    },
    filterCategories: function () {
      if (this.filterCategory) {
        const cats = this.$store.getters['categories/all_categories'];
        const isChild = p => c => c.length > p.length && _.isEqual(p, c.slice(0, p.length));
        const children = _.filter(cats, isChild(this.filterCategory));
        return [this.filterCategory].concat(children);
      } else {
        return null;
      }
    },
    link_prefix: function () {
      return `/activity/${this.host}/${this.periodLength}`;
    },
    periodusage: function () {
      return this.$store.getters['activity/getActiveHistoryAroundTimeperiod'](this.timeperiod);
    },
    timeperiod: function () {
      return { start: get_day_start_with_offset(this._date), length: [1, this.periodLength] };
    },
    dateformat: function () {
      if (this.periodLength === 'day') {
        return 'YYYY-MM-DD';
      } else if (this.periodLength === 'week') {
        return 'YYYY[ W]WW';
      } else if (this.periodLength === 'month') {
        return 'YYYY-MM';
      } else if (this.periodLength === 'year') {
        return 'YYYY';
      } else {
        return 'YYYY-MM-DD';
      }
    },
    periodReadable: function () {
      return moment(this.timeperiod.start).format(this.dateformat);
    },
    periodLengthMoment: function () {
      return this.periodLengthConvertMoment(this.periodLength);
    },
  },
  watch: {
    timeperiod: function () {
      this.refresh();
    },
    filterCategory: function () {
      this.refresh();
    },
  },

  mounted: async function () {
    this.$store.dispatch('views/load');
    await this.$store.dispatch('categories/load');
    await this.refresh();
  },

  methods: {
    previousPeriod: function () {
      return moment(this._date).subtract(1, `${this.periodLength}s`).format('YYYY-MM-DD');
    },
    nextPeriod: function () {
      return moment(this._date).add(1, `${this.periodLength}s`).format('YYYY-MM-DD');
    },
    periodLengthConvertMoment(periodLength) {
      if (periodLength === 'day') {
        return 'day';
      } else if (periodLength === 'week') {
        /* This is necessary so the week starts on Monday instead of Sunday */
        return 'isoWeek';
      } else if (periodLength === 'month') {
        return 'month';
      } else if (periodLength === 'year') {
        return 'year';
      } else {
        console.error('Invalid periodLength ${periodLength}, defaulting to "day"');
        return 'day';
      }
    },

    setDate: function (date, periodLength) {
      // periodLength is an optional argument, default to this.periodLength
      if (!periodLength) {
        periodLength = this.periodLength;
      }
      const new_period_length_moment = this.periodLengthConvertMoment(periodLength);
      const new_date = moment(date).startOf(new_period_length_moment).format('YYYY-MM-DD');
      console.log(new_date, periodLength);
      this.$router.push(`/activity/${this.host}/${periodLength}/${new_date}/${this.subview}`);
    },

    refresh: async function (force) {
      await this.$store.dispatch('activity/ensure_loaded', {
        timeperiod: this.timeperiod,
        host: this.host,
        force: force,
        filterAFK: true,
        filterCategories: this.filterCategories,
      });
    },

    load_demo: async function () {
      await this.$store.dispatch('activity/load_demo');
    },

    checkFormValidity() {
      // All checks must be false for check to pass
      const checks = {
        // Check if view id is unique
        'ID is not unique': this.$store.state.views.views.map(v => v.id).includes(this.new_view.id),
        'Missing name': this.new_view.name === '',
      };
      const errors = Object.entries(checks)
        .filter(([_k, v]) => v)
        .map(([k, _v]) => k);
      const valid = errors.length == 0;
      if (!valid) {
        alert(`Invalid form input: ${errors}`);
      }
      return valid;
    },

    handleOk(event) {
      // Prevent modal from closing
      event.preventDefault();
      // Trigger submit handler
      this.handleSubmit();
    },

    handleSubmit() {
      // Exit when the form isn't valid
      const valid = this.checkFormValidity();
      if (!valid) {
        return;
      }

      this.$store.commit('views/addView', { id: this.new_view.id, name: this.new_view.name });

      // Hide the modal manually
      this.$nextTick(() => {
        this.$refs.new_view.hide();
      });
    },

    resetModal() {
      this.new_view = {
        id: '',
        name: '',
      };
    },
    save() {
      this.$store.dispatch('views/save');
    },
    discard() {
      this.$store.dispatch('views/load');
    },
    remove() {
      this.$store.commit('views/removeView', { view_id: this.view.id });
      // If we're on an URL that'll be invalid after removing the view, navigate to the main/default view
      if (!this.$route.path.includes('default')) {
        this.$router.replace('./default');
      }
    },
    restoreDefaults() {
      this.$store.commit('views/restoreDefaults');
      alert(
        "All views have been restored to defaults. Changes won't be saved until you click 'Save'."
      );
      // If we're on an URL that might become invalid, navigate to the main/default view
      if (!this.$route.path.includes('default')) {
        this.$router.replace('./default');
      }
    },
    isVisLarge(el) {
      return el.type == 'sunburst_clock';
    },
  },
};
</script>
