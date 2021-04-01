<template>
    <div class="harbor-widget harbor-widget-search" :class="{ ['h-direction-' + direction]: true, 'h-expandable': expandable, 'h-initial-load': $apollo.queries.settings.loading, ['h-bar-theme-' + barTheme]: true }">
        <div class="h-expandable-bottom" v-if="expandable && additionalInformation">
            <div class="h-expand">
                <button class="h-button h-expand-button" @click="expanded = !expanded">{{ label ? label : t('Zoek in het aanbod') }}</button>

                <div class="h-details">
                    <div class="h-detail-period">
                        <span class="h-arrival">{{ formatHumanShort(selected.arrival) }}</span> <span class="h-divider"></span> <span class="h-departure">{{ formatHumanShort(selected.departure) }}</span>
                    </div>
                    <div class="h-detail-people">{{ countedPeople }}</div>
                </div>

                <button class="h-close" v-show="expanded" @click="open ? open = null : expanded = null"></button>
            </div>
        </div>

        <div class="h-expand" :class="{'h-open': expanded}" v-else-if="expandable">
            <button class="h-button h-expand-button" @click="expanded = !expanded">{{ label ? label : t('Zoek in het aanbod') }}</button>

            <button class="h-close" v-show="expanded" @click="open ? open = null : expanded = null"></button>
        </div>
        <div class="h-loading-dates-message" v-if="(initial || ($apollo.queries.availability.loading && config.availability)) && !error.all" style="display: none;">
          {{ t('Datums worden geladen') }}
        </div>
        <div class="h-redirect-messagee" v-if="redirectOverlay">
          {{ t('U wordt omgeleid, een ogenblik geduld') }}
        </div>
        <div class="h-search-extra" v-if="error.all">
          <div class="h-unauthenticated-message">
            {{ t('Helaas kunt u op dit moment niet online boeken') }}
          </div>
        </div>
        <div class="h-search-columns" :class="{ 'h-expanded': expanded, 'h-has-additional-information': additionalInformation, }" ref="columns" v-else>
            <div class="h-top-categories" v-if="((!settings.general && options.categories) && options.top) && (!given.category && !categoryId)">
                <ul>
                    <li v-for="(category, i) in filteredCategories" :class="{ 'h-selected': selected.category === category.id }" :key="i">
                        <a href="#" @click="$event.preventDefault(); select('category', category.id);">{{ category.title }}</a>
                    </li>
                </ul>
            </div>
            <div class="h-columns" v-if="!error.notFound">
                <div class="h-column" v-if="(((settings.general && !settings.filter && options.generalCategories) || (options.categories && !options.top)) || (expandable && expanded && options.categories)) && (!given.category && !categoryId)">
                    <div class="h-select-box h-select-categories" :class="{ open: open === 'categories' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('categories')">
                            <span class="h-label">{{ t('Categorie') }}</span>
                            <span class="h-value">{{ category && category.title ? category.title : (selected.category ? '' : t('Alle categorieën')) }}</span>
                        </button>
                        <div class="h-select-body">
                            <ul class="h-select-list">
                                <li v-if="settings.general">
                                    <a href="#" @click="$event.preventDefault(); select('category', null); open = null" v-translate>Alle categorieën</a>
                                </li>
                                <li v-for="(item, i) in filteredCategories" :key="i">
                                    <a href="#" @click="$event.preventDefault(); select('category', item.id); open = null">{{ item.title }}</a>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.accommodations && selected.category" v-show="options.displayAccommodations">
                    <div class="h-select-box h-select-accommodations" :class="{ open: open === 'accommodations' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('accommodations')">
                            <span class="h-label" v-translate>Accommodatie</span>
                            <span class="h-value">{{ accommodation ? accommodation.title : '-' }}</span>
                        </button>
                        <div class="h-select-body">
                            <ul class="h-select-list">
                              <template v-if="accommodationTheme !== null && mapWidget">
                                <div class="harbor-widget-overview h-widget-alternative">
                                  <div class="h-accommodations">
                                    <div class="h-columns">
                                      <div class="h-column" v-for="(item, i) in accommodations" :key="i" >
                                        <component :show-price="true" :is="'accommodation-' + accommodationTheme" :url="settings.url" :people="selected.people" :accommodation="item" :arrival="format(selected.arrival)" :departure="format(selected.departure)" :more-info="options.more" :show-person-price="false" :show-features="true" :for-map="true" @selected-accommodation="selected.accommodation = item.id; open = null; $emit('show-map')" />
                                      </div>
                                    </div>
                                  </div>
                                </div>
                              </template>
                              <li v-else v-for="(item, i) in accommodations" :key="i">
                                <a href="#" @click="$event.preventDefault(); selected.accommodation = item.id; open = null">{{ item.title }}</a>
                              </li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.filter && (settings.filter && settings.filter.type === 2)">
                    <div class="h-select-box h-select-filter h-select-filter-dependant" :class="{ open: open === 'filter' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('filter')">
                            <span class="h-label">{{ settings.filter.title }}</span>
                            <span class="h-value">{{ selected.filters[settings.filter.id] !== null ? settings.filter.choices[selected.filters[settings.filter.id]] : '' }}</span>
                        </button>
                        <div class="h-select-body">
                            <ul class="h-select-list">
                                <li v-for="(item, i) in settings.filter.choices" :key="i">
                                    <a href="#" @click="$event.preventDefault(); selected.filters[settings.filter.id] = i; open = null">{{ item }}</a>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="!!searchFilters">
                    <div class="h-select-box h-select-search-filter" :class="{ open: open === 'filter-custom'}" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('filter-custom')">
                            <span class="h-label">{{ customSearchFiltersLabel ? customSearchFiltersLabel : searchFiltersLabel }}</span>
                        </button>
                        <div class="h-select-body">
                            <ul class="h-select-list">
                                <li v-for="(filter, i) in availableSearchFilters" :key="i">
                                  <div class="h-filter-label">
                                    <label class="h-input-label">
                                      <input type="radio" class="h-input-radio" :value="true" v-model="selected.searchFilters[filter.id]" v-if="filter.type === 0">
                                      {{ t('Ja ik wil') }} {{ filter.title }}
                                    </label>
                                  </div>
                                  <div class="h-filter-label">
                                    <label class="h-input-label h-filter-label ">
                                      <input type="radio" class="h-input-radio" :value="false" v-model="selected.searchFilters[filter.id]" v-if="filter.type === 0">
                                      {{ t('Nee ik wil geen') }} {{ filter.title }}
                                    </label>
                                  </div>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.people">
                    <div class="h-select-box h-select-people" :class="{ open: open === 'people' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('people')">
                            <span class="h-value">{{ countedPeople }}</span>
                            <span class="h-label">{{ t(countedPeople != 1 ? 'personen' : 'persoon') }}</span>
                        </button>
                        <div class="h-select-body">
                            <div class="select-input-container" v-for="(value, i) in people" :key="i">
                                <label>{{ value.title }}</label>
                                <div class="select-input-group" :class="{ 'h-modifier-with-range': maximumPeople > 20 }">
                                    <input type="range" class="h-range" :min="(minPeople && minPeople[value.id] ? minPeople[value.id] : (value.minimum ? value.minimum : 0))" :max="maximumPeople" :value="selected.people[value.id]" @input="selected.people[value.id] = parseInt($event.target.value)" v-if="maximumPeople > 20" />

                                    <div class="h-modifiers">
                                        <button class="h-modifier h-modifier-minus" :class="{ 'h-disabled': selected.people[value.id] === 0 || countedPeople <= minimumPeople || (minPeople && minPeople[value.id] ? selected.people[value.id] <= minPeople[value.id] : (value.minimum ? selected.people[value.id] <= value.minimum : false)) || countedPeople <= 1 }" @click="((minPeople && minPeople[value.id] ? selected.people[value.id] > minPeople[value.id] : (value.minimum ? selected.people[value.id] > value.minimum : true)) && countedPeople > 1) ? selected.people[value.id]-- : null"></button>
                                        <input type="text" class="select-input" v-model="selected.people[value.id]">
                                        <button class="h-modifier h-modifier-plus" :class="{ 'h-disabled': countedPeople === maximumPeople || (value.max ? selected.people[value.id] >= value.max : false) }" @click="(countedPeople < maximumPeople && (value.max ? selected.people[value.id] < value.max : true)) ? selected.people[value.id]++ : null"></button>
                                    </div>
                                </div>
                            </div>

                            <div class="h-group-description h-people-information">
                                <div class="h-label">{{ t('Maximum personen') }}: <span class="h-value">{{ maximumPeople || '-' }}</span></div>
                                <button type="button" class="h-button" @click="open = null; silentUpdate()" v-translate>Gereed</button>
                            </div>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.filter && (settings.filter && settings.filter.type !== 2)">
                    <div class="h-select-box h-select-filter" :class="{ open: open === 'filter' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('filter')">
                            <span class="h-value">{{ countedFilter }}</span>
                            <span class="h-label">{{ settings.filter.singular && countedFilter === 1 ? settings.filter.singular : settings.filter.title }}</span>
                        </button>
                        <div class="h-select-body">
                            <div class="select-input-container">
                                <label>{{ settings.filter.title }}</label>
                                <div class="select-input-group" :class="{ 'h-modifier-with-range': settings.filter.maximum > 20 }">
                                    <input type="range" class="h-range" :min="0" :max="settings.filter.maximum" :value="selected.filters[settings.filter.id]" @input="selected.filters[settings.filter.id] = parseInt($event.target.value)" v-if="settings.filter.maximum > 20" />

                                    <div class="h-modifiers">
                                        <button class="h-modifier h-modifier-minus" :class="{ 'h-disabled': selected.filters[settings.filter.id] === 0 }" @click="(selected.filters[settings.filter.id] > 0) ? selected.filters[settings.filter.id]-- : null"></button>
                                        <input type="text" class="select-input" v-model="selected.filters[settings.filter.id]">
                                        <button class="h-modifier h-modifier-plus" :class="{ 'h-disabled': selected.filters[settings.filter.id] >= settings.filter.maximum }" @click="selected.filters[settings.filter.id] >= settings.filter.maximum ? null : selected.filters[settings.filter.id]++"></button>
                                    </div>
                                </div>
                            </div>

                            <div class="h-group-description h-filter-information">
                                <div class="h-label">{{ t('Maximum') }} {{ settings.filter.title }}: <span class="h-value">{{ settings.filter.maximum || '-' }}</span></div>
                                <button type="button" class="h-button" @click="open = null; silentUpdate()">{{ t('Gereed') }}</button>
                            </div>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.months || settings.months">
                    <div class="h-select-box h-select-months" :class="{ open: open === 'months' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('months')">
                            <span class="h-label">{{ t('Maand') }}</span>
                            <span class="h-value">{{ month ? formatMonth(month.month - 1, month.year) : '' }}</span>
                        </button>
                        <div class="h-select-body">
                            <ul class="h-select-list">
                                <li v-for="(item, i) in months" :class="{ 'h-disabled': !item.available }" :key="i">
                                    <a href="#" @click="$event.preventDefault(); item.available ? (selected.month = item) : null; open = null">{{ formatMonth(item.month - 1, item.year) }}</a>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.periods">
                    <div class="h-select-box h-select-periods" :class="{ open: open === 'period' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('period')">
                            <span class="h-label">{{ t('Periode') }}</span>
                            <span class="h-value">{{ period.title }}</span>
                        </button>
                        <div class="h-select-body">
                            <ul class="h-select-list">
                                <li v-for="(item, i) in filteredPeriods" :key="i">
                                    <a href="#" @click="$event.preventDefault(); selected.period = item.id; open = null">{{ item.title }}</a>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.packages">
                    <div class="h-select-box h-select-packages" :class="{ open: open === 'package' }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('package')">
                            <span class="h-value" v-if="pack">{{ packageLabel(pack) }}</span>
                            <span class="h-label" v-else>{{ t('Kies een verblijfsduur') }}</span>
                        </button>
                        <div class="h-select-body">
                            <ul class="h-select-list">
                                <li v-for="(item, i) in filteredPackages" :key="'packages-' + i">
                                    <a href="#" @click.prevent="selected.package = item.id; open = null; toggle('arrival')">{{ packageLabel(item) }}</a>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="options.arrival" v-show="options.showDates">
                    <div class="h-select-box h-select-arrival" :class="{ open: open === 'arrival', 'h-select-loading': initial || $apollo.queries.availability.loading, 'h-no-departure': !((!settings.periods && !options.periods) && options.departure) }" @click="checkClickOverlay($event)">
                        <button class="h-select-button" @click="toggle('arrival')">
                            <span class="h-label">{{ t('Aankomst') }}</span>
                            <span class="h-value">{{ selected.arrival ? formatHuman(selected.arrival) : '' }}</span>
                        </button>
                        <div class="h-select-body">
                            <div class="h-header">{{ t('Aankomstdag') }}</div>

                            <div class="h-loading-message" v-if="(initial || $apollo.queries.availability.loading) && !error.noDates">
                              {{ t('Datums worden geladen, een moment geduld') }}
                            </div>

                            <div class="h-dates-message" v-if="error.noDates">
                              {{ t('Er zijn helaas geen beschikbare datums gevonden') }}
                            </div>

                            <datepicker v-show="!(initial || $apollo.queries.availability.loading || error.noDates)" :occupied="dates.occupied" :packages="dates.packages" :disabled="dates.arrival.disabled" @selected="toggle('departure')" v-model="selected.arrival" ref="arrival" />

                            <ul class="h-legend">
                                <li class="h-legend-available" v-translate>Beschikbaar</li>
                                <li class="h-legend-highlighted" v-translate>Uw keuze</li>
                                <li class="h-legend-occupied" v-translate>Bezet</li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column" v-if="(!settings.periods && !options.periods) && options.departure" @click="checkClickOverlay($event)" v-show="options.showDates && options.showDeparture">
                    <div class="h-select-box h-select-departure" :class="{ open: open === 'departure' }">
                        <button class="h-select-button" @click="toggle('departure')">
                            <span class="h-label">{{ t('Vertrek') }}</span>
                            <span class="h-value">{{ selected.departure ? formatHuman(selected.departure) : '' }}</span>
                        </button>
                        <div class="h-select-body">
                            <div class="h-header">{{ t('Vertrekdag') }}</div>

                            <div class="h-loading-message" v-if="(initial || $apollo.queries.availability.loading) && !error.noDates">
                              {{ t('Datums worden geladen, een moment geduld') }}
                            </div>

                            <div class="h-dates-message" v-if="error.noDates">
                              {{ t('Er zijn helaas geen beschikbare datums gevonden') }}
                            </div>

                            <datepicker v-show="!(initial || $apollo.queries.availability.loading || error.noDates)" :occupied="dates.occupied" :highlighted="{ from: selected.arrival, to: selected.departure }" :disabled="dates.departure.disabled" @selected="open = null" v-model="selected.departure" ref="departure" />

                            <ul class="h-legend">
                                <li class="h-legend-available" v-translate>Beschikbaar</li>
                                <li class="h-legend-highlighted" v-translate>Uw keuze</li>
                                <li class="h-legend-occupied" v-translate>Bezet</li>
                            </ul>
                        </div>
                    </div>
                </div>
                <div class="h-column h-column-auto" v-if="!(config && !options.button)">
                    <div class="h-select-box">
                        <button class="h-button h-button-search" :class="{ 'h-loading': $apollo.loading }" @click="submit()">
                            {{ buttonLabel ? buttonLabel : t('Zoek') }}
                        </button>
                    </div>
                </div>
            </div>
            <div v-else class="h-error">
                {{ t('De accommodatie kon niet worden gevonden') }}
            </div>
        </div>
    </div>
</template>

<script>
  import Datepicker from './Datepicker';
  import metaQuery from '../queries/meta.gql';
  import periodsQuery from '../queries/periods.gql';
  import monthsQuery from '../queries/months.gql';
  import accommodationsQuery from '../queries/accommodations.gql';
  import accommodationQuery from '../queries/accommodation.gql';
  import categoriesQuery from '../queries/categories.gql';
  import filtersQuery from '../queries/filters.gql';
  import packagesQuery from '../queries/packages.gql';

  import isEqual from 'date-fns/is_equal';
  import isBefore from 'date-fns/is_before';
  import isAfter from 'date-fns/is_after';
  import isSameDay from 'date-fns/is_same_day';
  import addDays from 'date-fns/add_days';
  import subDays from 'date-fns/sub_days';
  import diffDays from 'date-fns/difference_in_calendar_days';
  import closestTo from 'date-fns/closest_to';
  import isValid from 'date-fns/is_valid';
  import eachDay from 'date-fns/each_day';
  import format from 'date-fns/format';
  import parse from 'date-fns/parse';
  import debounce from 'debounce';
  import bus from '../plugins/bus';
  import { parseQuery, getQueryString } from '../plugins/request';

  export default {
    props: {
      config: {
        type: Object,
        default() {
          return {
            generalCategories: true,
            categories: true,
            arrival: true,
            button: true,
          };
        },
      },

      direction: {
        type: String,
        default: 'down',
        validator(value) {
          switch (value) {
            case 'up':
            case 'down':
            case 'auto':
              return true;
          }

          return false;
        },
      },

      label: {
        type: String,
        default: null,
      },

      buttonLabel: {
        type: String,
        default: null,
      },

      expandable: {
        type: Boolean,
        default: false,
      },

      strict: {
        type: Boolean,
        default: false,
      },

      load: {
        type: Boolean,
        default: false,
      },

      preFill: {
        type: Boolean,
        default: false,
      },

      url: {
        type: String,
        default: null,
      },

      global: {
        type: Boolean,
        default: false,
      },

      categoryId: {
        type: Number,
        default: null,
      },

      periodId: {
        type: [Number, Array],
        default: null,
      },

      given: {
        type: Object,
        default() {
          return {
            category: null,
            accommodation: null,
            period: null,
          };
        },
      },

      filtered: {
        type: Array,
        default: null,
      },

      filters: {
        type: Object,
        default: null,
      },

      minPeople: {
        type: Object,
        default: null,
      },

      defaultPeople: {
        type: Object,
        default: null,
      },

      overridePeople: {
        type: Object,
        default: null,
      },

      overrideRememberFilters: {
        type: Boolean,
        default: false,
      },

      dateFormat: {
        type: Function,
        default: null,
      },

      mapWidget: {
        type: Boolean,
        default: false,
      },

      accommodationTheme: {
        type: String,
        default: 'normal',
        validator(value) {
          return ['normal', 'alternative'].indexOf(value) !== -1
        }
      },

      searchFilters: {
        type: Object,
        default: null
      },

      searchCategory: {
        type: Number,
        default: null,
      },

      customSearchFiltersLabel: {
        type: String,
        default: null
      },

      additionalInformation: {
        type: Boolean,
        default: false
      },

      barTheme: {
        type: String,
        default: 'default',
        validator(value) {
          return ['default', 'white'].indexOf(value) !== -1
        }
      },

      flexiblePackages: {
        type: Boolean,
        default: false
      },

      packageCategory: {
        type: Number,
        default: null
      },

      packageLabel: {
        type: Function,
        default: function (item) {
          if (!item) {
            return '';
          }

          return item.title;
        }
      }
    },

    components: {
      datepicker: Datepicker,
      accommodationNormal: require('./Accommodation'),
      accommodationAlternative: require('./AccommodationAlt'),
    },

    apollo: {
      settings: {
        query: metaQuery,

        result({ data, error }) {
          if (error !== void 0 && error.networkError && error.networkError.result && error.networkError.result.error == 'Unauthenticated.') {
            this.error.all = true;

            return;
          }

          if (data.settings.locales) {
            this.$translate.validate(data.settings.locales);
          }

          if (data.settings.filter) {
            this.$set(this.selected.filters, data.settings.filter.id, this.selected.filters[data.settings.filter.id] === void 0 ? (data.settings.filter.default || 0) : this.selected.filters[data.settings.filter.id]);

            if (this.load && this.initial) {
              this.$nextTick(() => {
                this.update();

                this.initial = false
              });
            }
          }
        },
      },

      people: {
        query: metaQuery,

        result({ data, error }) {
          if (error !== void 0) {
            return;
          }

          const people = {};
          const hasDefault = data.people.find((item) => (
            item.default > 0
          ));

          data.people.forEach((item, i) => {
            if (this.overridePeople) {
              people[item.id] = (this.overridePeople && this.overridePeople[item.id]) || item.default || (i === 0 && !hasDefault ? 2 : 0) || 0
            } else {
              people[item.id] = this.selected.people[item.id] === void 0 ? (i === 0 && !hasDefault ? 2 : ((this.defaultPeople && this.defaultPeople[item.id] && this.defaultPeople[item.id]) || item.default || 0)) : this.selected.people[item.id]
            }
          });

          this.$set(this.selected, 'people', people);

          this.silentUpdate();
        },
      },

      periods: {
        query: periodsQuery,

        variables() {
          return {
            category: this.selected.category,
            accommodation: this.selected.accommodation,
          };
        },

        skip() {
          if (this.error.all) {
            return true;
          }

          if (!(this.settings.periods || this.options.periods)) {
            return true;
          }

          if (this.$apollo.queries.settings.loading) {
            return true;
          }

          return false;
        },

        result({ data }) {
          if (!this.preFill) {
            return;
          }

          if (!this.options.arrival && !this.options.departure) {
            const period = data.periods.find((item) => (
              item.nights == diffDays(this.selected.departure, this.selected.arrival)
            ))

            if (period) {
              this.selected.period = period.id;
            }
          }

          if (!this.selected.period) {
            this.selected.period = this.filteredPeriods[0].id;
          } else {
            const period = this.filteredPeriods.find((period) => (
              period.id === parseInt(this.selected.period)
            ));

            if (!period) {
              this.selected.period = this.filteredPeriods[0].id;
            }
          }

          if (!this.options.arrival && (this.preFill && this.initial)) {
            this.$nextTick(() => {
              this.update();

              this.initial = false
            });
          }
        },
      },

      months: {
        query: monthsQuery,

        variables() {
          return {
            category: this.selected.category,
            accommodation: this.selected.accommodation,
            period: (this.settings.periods || this.options.periods) ? this.selected.period : null,
          };
        },

        skip() {
          if (this.error.all) {
            return true;
          }

          if (!(this.settings.months || this.options.months)) {
            return true;
          }

          if ((this.settings.periods || this.options.periods) && !this.selected.period) {
            return true;
          }

          return false;
        },
      },

      categories: {
        query() {
          return metaQuery;
        },

        result({ data, error }) {
          if (error !== void 0) {
            return;
          }

          const category = data.categories.find((category) => (
            category.id === parseInt(this.selected.category)
          ));

          if (!category && this.options.categories) {
            this.select('category', null);
          }

          if (!this.settings.general && !this.selected.category && !this.strict) {
            if (this.filteredCategories && this.filteredCategories.length > 0) {
              this.select('category', this.filteredCategories[0].id);
            }
          }

          this.$storage.saveSession('categories', data.categories);
        },
      },

      allCategories: {
        query() {
          return categoriesQuery;
        },

        skip() {
          if (this.error.all) {
            return true;
          }

          return !this.settings.filter || this.settings.filter.type !== 2;
        },

        update: data => data.categories,
      },

      accommodations: {
        query: accommodationsQuery,

        variables() {
          return {
            category: this.selected.category,
          };
        },

        skip() {
          return !this.shouldLoadAccommodations;
        },

        result({ data }) {
          this.$storage.saveSession('accommodations', data.accommodations);

          if (!this.selected.accommodation) {
            this.selected.accommodation = data.accommodations[0].id;
          } else {
            const accommodation = data.accommodations.find((accommodation) => (
              accommodation.id === this.selected.accommodation
            ));

            if (!accommodation && data.accommodations.length > 0) {
              this.selected.accommodation = data.accommodations[0].id;
            }
          }
        },
      },

      accommodation: {
        query: accommodationQuery,

        variables() {
          return {
            id: this.selected.accommodation,
            withPackages: this.options.packages
          };
        },

        skip() {
          if (this.error.all) {
            return true;
          }

          if (!this.options.accommodations && !this.options.accommodation) {
            return true;
          }

          return !(!!this.selected.accommodation && !this.selected.category);
        },

        manual: true,

        result ({ data, loading }) {
          if (!loading) {
            if (!data.accommodation) {
              this.error.notFound = true;

              return;
            }

            this.accommodations = [data.accommodation];
          }
        },
      },

      availability: {
        query: require('../queries/availability.gql'),

        context: {
          batching: false,
        },

        variables() {
          const people = [];

          Object.keys(this.selected.people).forEach((id) => {
            people.push({
              id,
              value: this.selected.people[id],
            });
          });

          return {
            category: this.selected.category,
            accommodation: this.options.accommodations ? this.selected.accommodation : null,
            period: this.options.periods ? this.selected.period : null,
            people,
            packageCategory: this.packageCategory,
            package: this.options.packages ? this.selected.package : null
          };
        },

        skip() {
          if (this.error.all) {
            return true;
          }

          /**
           * If the availability paramater is manually set to false, skip.
           */
          if (this.config.availability === false) {
            this.initial = false;

            return true;
          }

          /**
           * General is that no category needs to be selected.
           */
          if (!this.settings.general && !this.selected.category && !this.strict) {
            return true;
          }

          /**
           * Get the information needed to built the calendars.
           */
          if (this.$apollo.queries.settings.loading) {
            return true;
          }

          /**
           * If the periods setting is enabled, check if the periods are done loading.
           */
          if ((this.settings.periods || this.options.periods) && (this.$apollo.queries.periods.loading || !this.selected.period)) {
            return true;
          }

          /**
           * If a accommodation is given, check if the accommodations are done loading.
           */
          if (this.selected.accommodation && this.$apollo.queries.accommodations.loading) {
            return true;
          }

          /**
           * If the arrival and periods are disabled, no availability needs to be loaded.
           */
          if (!this.options.arrival && (this.settings.periods || this.options.periods)) {
            return true;
          }

          /**
           * If the arrival and departure are disabled, no availability needs to be loaded.
           */
          if (!this.options.arrival && !this.options.departure) {
            return true;
          }

          if (!this.selected.category && !this.selected.accommodation && !this.settings.general) {
            return true;
          }

          /**
           * When too many inc/decrements are made to the total people, debounce.
           */
          if (this.debouncing) {
            return true;
          }

          /**
           * If there is an error, skip.
           */
          if (this.error.notFound) {
            return true;
          }

          return false;
        },

        result({ data, loading }) {
          if (!loading) {
            this.$nextTick(() => {
              if (this.unlockSilentUpdate) {
                this.unlockSilentUpdate = false;
                this.silentUpdate();
              }
            });
          }
        },
      },

      allFilters: {
        query: filtersQuery,

        update: data => data.filters,

        skip() {
          return !this.searchFilters;
        }
      },

      packages: {
        query: packagesQuery,

        variables() {
          const people = [];

          Object.keys(this.selected.people).forEach((id) => {
            people.push({
              id,
              value: this.selected.people[id],
            });
          });

          let packages = [];

          if (this.accommodation.packages) {
            packages = this.accommodation.packages.filter(p => {
              if (this.flexiblePackages) {
                return p.flexible;
              }

              return true;
            }).map(p => p.id);
          }

          return {
            arrival: this.format(this.selected.arrival ? this.selected.arrival : this.firstArrival),
            departure: this.format(this.selected.departure),
            packages,
            people,
          }
        },

        skip() {
          if (this.packagesLoaded) {
            return true;
          }

          if (!this.options.packages) {
            return true;
          }

          if (!this.selected.people || (this.selected.people && !Object.values(this.selected.people).length)) {
            return true;
          }

          if (this.$apollo.queries.settings.loading) {
            return true;
          }

          if (!(this.selected.arrival || this.firstArrival)) {
            return true;
          }

          return false;
        },

        result({data}) {
          if (this.selected.packageDuration && this.flexiblePackages) {
            const found = this.filteredPackages.find(pack => pack.days === this.selected.packageDuration);

            if (found) {
              this.selected.package = found.id;
            }
          }

          this.$nextTick(() => {
            this.packagesLoaded = true;
          })
        }
      }
    },

    data() {
      return {
        initial: true,
        debouncing: false,
        unlockSilentUpdate: false,
        redirectOverlay: false,

        settings: {
          periods: false,
          general: false,
          filter: {
            id: null,
          },
        },

        error: {
          notFound: false,
          noDates: false,
          all: false,
        },

        people: [],

        categories: this.$storage.retrieveSession('categories') || [],

        allCategories: [],

        accommodations: this.shouldLoadAccommodations ? (this.$storage.retrieveSession('accommodations') || []) : [],

        periods: [],

        months: [],

        selected: {
          category: this.given.category || this.categoryId,
          searchCategory: null,
          accommodation: this.given.accommodation,
          arrival: this.given.arrival || null,
          departure: this.given.departure || null,
          period: this.given.period,
          month: null,
          people: {},
          filters: this.filters || {},
          searchFilters: this.searchFilters || {},
          filter: null,
          hasQuery: false,
          package: null,
          packageDuration: null
        },

        periodDate: {},

        previous: {},

        open: null,

        expanded: false,

        availability: [],

        dates: {
          arrival: {
            disabled: {
              dates: [],
              from: null,
              to: null,
            },
          },

          departure: {
            disabled: {
              dates: [],
              from: null,
              to: null,
            },
          },

          occupied: {
            dates: [],
          },

          packages: {
            dates: [],
          },
        },

        dontUpdateSelectedMonth: false,

        allFilters: [],

        packages: [],

        firstArrival: null,

        packagesLoaded: false
      };
    },

    watch: {
      'availability'() {
        const dates = this.availability;

        if (!dates) {
          this.initial = false;
          this.error.noDates = true;
          this.selected.arrival = null;
          this.selected.departure = null;

          this.$emit('noDates', true);

          return;
        }

        const firstArrival = this.availability.find((date) => date.arrival)
        // const firstDeparture = this.availability.find((date) => date.departure)

        if (!firstArrival) {
          this.initial = false;
          this.error.noDates = true;
          this.selected.arrival = null;
          this.selected.departure = null;

          this.$emit('noDates', true);

          return;
        }

        this.error.noDates = false;

        const previousSelected = {
          ...this.selected,
        };

        const arrival = this.getProbableArrivalDate(dates, false, new Date);

        if (arrival) {
          this.dates.arrival.disabled.to = arrival.date;

          if ((!this.selected.arrival || (this.selected.arrival && isAfter(arrival.date, this.selected.arrival))) && this.$refs.arrival) {
            this.$refs.arrival.setCurrentMonth(arrival.date);
          }

          this.$emit('first-arrival', this.firstArrival = arrival.date);
        }

        if (this.selected.arrival) {
          const arrival = this.getProbableArrivalDate(dates, false, this.format(this.selected.arrival));

          if (arrival) {
            if (!isEqual(arrival.date, this.selected.arrival)) {
              this.selected.arrival = parse(arrival.date);
            }

            this.dates.departure.disabled.to = this.format(
              addDays(this.selected.arrival, arrival.minimum_duration || this.category.minimum_days || 1)
            );
          } else {
            const found = this.getProbableArrivalDate(dates);

            this.selected.arrival = found ? found.date : null;
          }

        } else {
          const departure = dates.find((date) => (
            date.departure && isAfter(date.date, new Date)
          ));

          if (departure) {
            this.dates.departure.disabled.to = departure.date;
            this.$refs.departure.setCurrentMonth(departure.date);
          }
        }

        if (this.selected.departure) {
          const departure = dates.find((date) => (
            date.departure && date.date == this.format(this.selected.departure)
          ));

          if (!departure) {
            const found = dates.find((date) => (
                date.departure && isAfter(date.date, this.selected.arrival)
            ));

            this.selected.departure = found ? found.date : null;
          }
        } else {
          const departure = dates.find((date) => (
            date.departure && isAfter(date.date, this.selected.arrival)
          ));

          if (departure && this.$refs.departure) {
            this.$refs.departure.setCurrentMonth(departure.date);
          }
        }

        if (this.preFill && this.initial) {
          if (!this.selected.arrival) {
            const found = this.getProbableArrivalDate(dates, true, this.format(previousSelected.arrival));

            if (!found) {
              const date = dates.find((date) => (
                date.arrival && isAfter(date.date, new Date)
              ))

              if (date) {
                this.selected.arrival = parse(date.date);
              }
            } else {
              this.selected.arrival = parse(found.date);
            }
          }

          if (this.load) {
            this.update();
          } else {
            this.$nextTick(() => (
              this.silentUpdate()
            ));
          }

          this.$nextTick(() => (
            this.initial = false
          ));
        } else if (this.unlockSilentUpdate) {
          this.unlockSilentUpdate = false;

          this.$nextTick(() => (
            this.silentUpdate()
          ));
        }

        this.dates.arrival.disabled.dates = dates.filter((date) => !date.arrival).map((date) => (
          date.date
        ));

        this.dates.occupied.dates = dates.filter((date) => date.occupied && (!date.arrival && !date.departure)).map((date) => (
          parse(date.date)
        ));

        this.dates.packages.dates = dates.filter((date) => date.package).map((date) => (
          parse(date.date)
        ));

        this.dates.departure.disabled.dates = dates.filter((date) => !date.departure).map((date) => (
          date.date
        ));

        const lastDate = this.availability.slice(0).reverse();

        const lastArrival = lastDate.find((date) => (
          date.arrival
        ));

        this.dates.arrival.disabled.from = lastArrival ? parse(lastArrival.date) : null;

        const lastDeparture = lastDate.find((date) => (
          date.departure
        ));

        this.dates.departure.disabled.from = lastDeparture ? parse(lastDeparture.date) : null;

        if (this.selected.arrival && this.$refs.departure) {
          const currentArrival = this.availability.find((date) => (
            isSameDay(this.selected.arrival, date.date)
          ));
          const maxDays = currentArrival.maximum_duration || this.category.maximum_days || 0;
          const maxDeparture = addDays(
            this.selected.arrival, maxDays
          );

          const lastDate = this.availability.find((date) => (
            isAfter(date.date, this.selected.arrival) && (date.occupied || (maxDays > 0 ? isEqual(date.date, maxDeparture) : false))
          ));

          this.dates.departure.disabled.from = lastDate ? parse(lastDate.date) : parse(lastDeparture.date);

          if (this.selected.departure && (isAfter(this.selected.departure, this.dates.departure.disabled.from) || isBefore(this.selected.departure, this.dates.departure.disabled.to))) {
            const arrival = dates.find((date) => (
              date.arrival && date.date == this.format(this.selected.arrival)
            ));

            const minimumDays = Math.max(arrival.minimum_duration, this.accommodation.minimum_days, this.category.minimum_days);

            const departure = dates.find((date) => (
              date.departure && isAfter(date.date, addDays(this.selected.arrival, (minimumDays ? (minimumDays - 1 ) : null) || 1))
            ));

            this.selected.departure = departure ? parse(departure.date) : null;
          }
        }

        if (this.preFill && (!this.selected.arrival || !this.selected.departure)) {
          if (arrival && !this.selected.arrival) {
            this.selected.arrival = parse(arrival.date);
          }

          const departure = dates.find((date) => (
            date.departure && isAfter(date.date, this.selected.arrival)
          ));

          if (departure && !this.selected.departure) {
            this.selected.departure = parse(departure.date);
          }
        }

        /**
         * If it does not need to be pre-fillt, and is also the initial load, and also
         * the load prop is set to true, send an update.
         */
        if ((!this.preFill && this.initial) && this.load) {
          this.update();

          this.$nextTick(() => (
            this.initial = false
          ));
        } else {
          this.$nextTick(() => (
            this.initial = false
          ));
        }

        this.updateDisabledDepartureDates();
        this.updatePeriod();

        this.debouncing = false;
      },

      'months'() {
        if (!this.preFill) {
          return;
        }

        if (this.selected.arrival) {
          const date = parse(this.selected.arrival);

          if (this.dontUpdateSelectedMonth) {
            this.dontUpdateSelectedMonth = false;
          } else if (this.selected.month) {
            const available = !!this.months.find((month) => month.month == this.selected.month.month);

            if (available && (this.selected.month.year != date.getFullYear() || this.selected.month.month != (date.getMonth() + 1))) {
              this.selected.month = {
                month: date.getMonth() + 1,
                year: date.getFullYear(),
              };
            }
          }

        }

        if (!this.selected.month) {
          this.selected.month = this.months.find((month) => month.available);
        } else {
          const month = this.months.find((month) => (
            month.month === this.selected.month.month && month.year === this.selected.month.year && month.available
          ));

          if (!month) {
            this.selected.month = this.months.find((month) => month.available);
          }
        }

        if ((!this.options.arrival && !this.options.departure) && (this.preFill)) {
          this.update();

          this.$nextTick(() => (
            this.initial = false
          ));
        }
      },

      'selected.arrival'(item) {
        if (this.error.noDates) {
          return;
        }

        const selectedArrival = this.availability.find((date) => (
          isEqual(date.date, this.selected.arrival)
        ));

        if (selectedArrival !== void 0) {
          const minimumDays = Math.max(selectedArrival.minimum_duration || 0, this.category.minimum_days || 0, this.accommodation.minimum_days || 0);

          this.dates.departure.disabled.to = this.format(
            item = addDays(item, minimumDays)
          );
        }

        const firstDepartureDate = this.availability.find((date) => (
          date.departure && (isAfter(date.date, item) || isSameDay(date.date, item))
        ));

        if (firstDepartureDate) {
          if (this.$refs.departure) {
            this.$nextTick(() => {
              this.$refs.departure.setCurrentMonth(firstDepartureDate.date);
            });
          }
        }

        if ((isAfter(item, this.selected.departure) || isAfter(this.dates.departure.disabled.to, this.selected.departure)) && this.$refs.departure) {
          const parsed = (item && this.selected.departure) ? this.availability.find((date) => (
            date.departure && (isAfter(date.date, item) || isSameDay(date.date, item))
          )) : null;

          this.selected.departure = parsed ? parse(parsed.date) : null;
        }

        if (this.availability && this.availability.length > 0 && typeof selectedArrival !== "undefined") {
          let lastDate = this.availability.find((date) => (
            (date.occupied || isEqual(date.date, addDays(
              this.selected.arrival, selectedArrival.maximum_duration || this.category.maximum_days || 0
            ))) && isAfter(date.date, this.selected.arrival)
          ));

          if (!lastDate) {
            const reversed = this.availability.slice(0).reverse();

            lastDate = reversed.find((date) => (
              date.departure
            ));
          }

          this.dates.departure.disabled.from = lastDate ? parse(lastDate.date) : null;

          if (isAfter(this.selected.departure, this.dates.departure.disabled.from)) {
            const parsed = item ? this.availability.find((date) => (
              date.departure && isAfter(date.date, item)
            )) : null;

            this.selected.departure = parsed ? parse(parsed.date) : null;
          }

          if (this.global && item) {
            const date = parse(item);

            this.selected.month = {
              month: date.getMonth() + 1,
              year: date.getFullYear(),
            };
          }

          this.updateDisabledDepartureDates();
        }
      },

      'selected.departure'() {
        if (this.availability) {
          this.updatePeriod();
        }
      },

      'selected.people': {
        deep: true,
        handler() {
          this.checkPeopleRestriction();

          if (!this.initial) {
            this.debouncing = true;
            // this.debouncePeople();
          }
        },
      },

      'selected.filters': {
        deep: true,
        handler() {
          let count = 0;

          Object.keys(this.selected.filters).forEach((id) => {
            let total = this.selected.filters[id];

            if (total < 0) {
              this.selected.filters[id] = 0;
            }

            if (total > this.settings.filter.maximum) {
              this.selected.filters[id] = (count === 0 ? (this.settings.filter.maximum || total) : 0);
            }

            if (count > this.settings.filter.maximum) {
              this.selected.filters[id] = 0;
            }

            count += total;
          });
        }
      },

      'selected.category'() {
        if (!this.category) {
          return;
        }

        this.checkPeopleRestriction();
      },

      'selected.accommodation'() {
        if (!this.accommodation) {
          return;
        }

        this.checkPeopleRestriction();
      },

      'selected.period'(newVal, oldVal) {
        if (oldVal) {
          this.dontUpdateSelectedMonth = true;
        }
      },

      'selected.package'() {
        if (this.pack) {
          this.selected.packageDuration = this.pack.days;
        }
      },

      'selected': {
        deep: true,
        handler() {
          if (this.initial || this.$apollo.queries.availability.loading) {
            return;
          }

          this.$nextTick(() => {
            if (this.$apollo.queries.availability && this.$apollo.queries.availability.loading) {
              this.unlockSilentUpdate = true;

              return;
            }

            this.silentUpdate();
          });
        },
      },

      'open'() {
        if (this.open !== 'people') {
          this.debouncing = false;
        }

        // if (this.open && this.direction === 'auto') {
        //   this.$nextTick(() => {
        //     const selectBox = this.$refs.columns.querySelector('.h-select-box.open .h-select-body');

        //     console.log(selectBox.getBoundingClientRect());

        //     console.log(
        //       selectBox.offsetParent.offsetHeight - selectBox.offsetTop,
        //       selectBox.getBoundingClientRect()
        //     );

        //     if (selectBox.offsetParent.offsetHeight - selectBox.offsetTop <= selectBox.offsetHeight) {

        //     }

        //     console.dir(selectBox);
        //   });
        // }

        this.$emit('field', this.open);
      },
    },

    computed: {
      countedPeople() {
        return Object.values(this.selected.people).reduce((a, b) => parseInt(a) + parseInt(b), 0);
      },

      countedFilter() {
        return Object.values(this.selected.filters).reduce((a, b) => parseInt(a) + parseInt(b), 0);
      },

      minimumPeople() {
        if (this.accommodation.id) {
          return this.accommodation.minimum_people || 1;
        }

        return 1;
      },

      maximumPeople() {
        if (this.accommodation.id) {
          return this.accommodation.maximum_people;
        }

        if (this.category.id) {
          return this.category.maximum_people;
        }

        let categories = this.categories;

        if (this.settings.filter && this.settings.filter.type === 2 && this.allCategories.length > 0) {
          categories = this.allCategories.map((category) => {
            category = {
              ...category,
              accommodations: category.accommodations.filter((accommodation) => {
                const found = accommodation.filters.find((filter) => (
                  this.settings.filter.id === filter.filter && filter.value == this.selected.filters[this.settings.filter.id]
                ));

                return !!found;
              }),
            }

            return category;
          });

          categories = categories.filter((category) => (
            category.accommodations.length > 0
          ));

          categories = categories.flatMap((category) => (
            category.accommodations
          ));
        }

        const max = Math.max.apply(
          Math, categories.map((o) => o.maximum_people)
        );

        if (max <= 0) {
          return 0;
        }

        return max;
      },

      filteredCategories() {
        if (this.filtered) {
          return this.categories.filter((category) => (
            this.filtered.indexOf(category.id) !== -1
          ));
        }

        return this.categories;
      },

      filteredPeriods() {
        if (this.periodId && Array.isArray(this.periodId)) {
          return this.periods.filter((period) => (
            this.periodId.indexOf(period.id) !== -1
          ));
        }

        return this.periods;
      },

      category() {
        const category = this.categories.find((category) => (
          parseInt(this.selected.category) === category.id
        ));

        if (!category) {
          return {};
        }

        return category;
      },

      accommodation() {
        const accommodation = this.accommodations.find((accommodation) => (
          this.selected.accommodation === accommodation.id
        ));

        if (!accommodation) {
          return {
            title: '-',
          };
        }

        return accommodation;
      },

      period() {
        const period = this.periods.find((period) => (
          parseInt(this.selected.period) === period.id
        ));

        if (!period) {
          return {};
        }

        return period;
      },

      month() {
        return this.selected.month || null;
      },

      options() {
        const options = {
            accommodations: false,
            accommodation: false,
            top: true,
            categories: true,
            generalCategories: true,
            periods: this.config.periods === void 0 ? this.settings.periods : this.config.periods,
            months: false,
            arrival: true,
            departure: true,
            button: true,
            filter: false,
            prices: true,
            people: true,
            showDates:true,
            displayAccommodations: true,
            packages: false,
            showDeparture: true,
        };

        const config = {
          ...options,
          ...Object.keys(this.config).filter((key) => this.config[key] !== void 0).reduce((obj, key) => {
            obj[key] = this.config[key];
            return obj;
          }, {}),
        };

        if (this.settings.filter && this.settings.filter.id && this.config.filter !== false) {
          config.arrival = false;
          config.departure = false;
          config.categories = false;
          config.filter = true;
          this.initial = false;
        }

        return config;
      },

      availableSearchFilters() {
        if (!this.searchFilters) {
          return [];
        }

        const filterIds = Object.keys(this.searchFilters).map(a => parseInt(a));

        return this.allFilters.filter(filter => {
          return filterIds.includes(filter.id)
        });
      },

      searchFiltersLabel() {
        if (!this.searchFilters) {
          return;
        }

        const filterIds = Object.keys(this.selected.searchFilters).filter(id => {
          return this.selected.searchFilters[id];
        }).map(a => parseInt(a));

        return this.availableSearchFilters.map(f => f.title).join(', ');
      },

      shouldLoadAccommodations() {
        if (this.error.all) {
          return false;
        }

        if (!this.options.accommodations) {
          return false;
        }

        return !!this.selected.category;
      },

      pack() {
        return this.filteredPackages.find(p => parseInt(p.id) === this.selected.package);
      },

      filteredPackages() {
        return (this.packages || []).filter(p => {
          if (this.flexiblePackages) {
            return p.flexible;
          }

          return true;
        }).filter(p => {
          if (this.packageCategory) {
            return p.categories.map(c => c.id).includes(this.packageCategory);
          }

          return true;
        });
      }
    },

    created() {
      let session = {
        ...this.selected,
        ...this.$storage.retrieve('selected'),
        hasQuery: false,
      };

      if (this.overrideRememberFilters) {
        session.filters = {}
      }

      if (this.filters != null) {
        session.filters = {
          ...session.filters,
          ...this.filters
        }
      }

      if (this.searchFilters != null) {
        if (!session.searchFilters) {
          session.searchFilters = {};
        }

        const searchFilters = Object.keys(this.searchFilters);

        Object.keys(session.searchFilters).forEach(id => {
          if (!searchFilters.includes(id)) {
            delete session.searchFilters[id];
          }
        });
      }

      if (this.config.availability === false) {
          this.dates.arrival.disabled.to = this.dates.departure.disabled.to = subDays(new Date, 1);
      }

      if (!session.timestamp || isBefore(session.timestamp, subDays(new Date, 1))) {
        this.$storage.save('selected', {
          timestamp: new Date,
        });

        this.$storage.save('ie_message_dismissed', false);

        session = {
          ...this.selected,
          timestamp: new Date,
        }
      }

      if (this.global) {
        if (bus.initial) {
          bus.initial = false;

          session = {
            ...bus.search,
            ...session,
          }
        } else {
          session = {
            ...session,
            ...bus.search,
            people: {
              ...bus.search.people,
            },
          }
        }
      }

      let explicitAttribute = false;

      if (this.categoryId) {
        session.searchCategory = parseInt(this.categoryId)
        session.category = parseInt(this.categoryId)

        explicitAttribute = true;
      }

      if (this.searchCategory) {
        session.searchCategory = parseInt(session.searchCategory || this.searchCategory)
        session.category = parseInt(session.category || this.searchCategory)
      }

      // Check the query
      const query = parseQuery(window.location.search);

      if (query != void 0) {
        let queryFilters = [];

        if (query.h_filters) {
          const filters = query.h_filters.split(',');

          if (filters && Array.isArray(filters)) {
            filters.forEach(filter => {
              queryFilters[filter] = true;
            });
          }

          delete query['h_filters'];
        }

        session.queryFilters = {
          ...queryFilters
        };

        if (query.h_category && !isNaN(query.h_category)) {
          session.category = parseInt(query.h_category);

          session.hasQuery = true;

          delete query['h_category'];
        }

        if (query.h_arrival) {
          const arrival = parse(query.h_arrival);
          const departure = parse(query.h_departure);

          session.hasQuery = true;

          if (query.h_accommodation && !isNaN(query.h_accommodation)) {
            session.accommodation = parseInt(query.h_accommodation);
          }

          if (query.h_period && !isNaN(query.h_period)) {
            session.period = query.h_period;
          }

          const people = {
            ...session.people,
          }

          if (isValid(arrival)) {
            session.arrival = arrival;
          }

          if (isValid(departure)) {
            session.departure = departure;
          }

          Object.keys(query).forEach((item) => {
            const list = /(.*?(\[(.*)\]))/g.exec(item);

            if (!list) {
              return;
            }

            if (list[3] && !isNaN(query[item])) {
              people[list[3]] = parseInt(query[item])
            }

            delete query[item];
          });

          session.people = people;

          delete query['h_arrival'];
          delete query['h_departure'];
          delete query['h_accommodation'];
          delete query['h_period'];
        }

        // Clean the empty key for query
        Object.keys(query).forEach(item => {
          if (!item) {
            delete query[item];
          }
        });

        history.replaceState({}, null, (
          window.location.origin + window.location.pathname + (Object.keys(query).length > 0 ? '?' + getQueryString(query) : '')
        ));
      }

      this.previous = Object.assign({}, session);

      const parsed = parse(session.arrival);

      Object.keys(session).forEach((key) => {
        if (
          (key === 'month' && !session[key]) ||
          (session[key] && (session[key].year != parsed.getFullYear() || session[key].month != (parsed.getMonth() + 1)))
        ) {
          session.month = session.arrival ? {
            month: parsed.getMonth() + 1,
            year: parsed.getFullYear(),
          } : null;
        }

        if (key === 'category' || key === 'accommodation') {
          if (!this.global) {
            if (!this.strict) {
              if (this.selected[key]) {
                return;
              }

              this.selected[key] = session[key];
            } else {
              this.selected[key] = this.selected[key] || ((session.hasQuery || (this.options.accommodations && this.selected.category) || (this.options.months && this.options.periods)) ? session[key] : null);
            }
          } else if (!this.strict) {
            bus.search[key] = session[key];
          } else {
            bus.search[key] = bus.search[key] || this.selected[key] || ((session.hasQuery || (this.options.accommodations && this.selected.category) || (this.options.months && this.options.periods)) ? session[key] : null);
          }

          return;
        }

        if (key === 'people' && this.global) {
          Object.keys(session[key]).forEach((value, person) => {
            bus.search[key][person] = value;
          });
        }

        if (key === 'filter' && this.global && session[key]) {
          Object.keys(session[key]).forEach((value, filter) => {
            bus.search[key][filter] = value;
          });
        }

        if (this.global) {
          bus.search[key] = session[key];
        } else {
          if (key === 'searchCategory' && (!session.hasQuery && !explicitAttribute)) {
            this.select('category', session[key]);
            return;
          }
          this.selected[key] = session[key];
        }
      });

      if (this.global) {
        this.selected = bus.search;
      }

      this.$nextTick(() => (
        this.silentUpdate()
      ));
    },

    methods: {
      formatHuman(date) {
        if (this.dateFormat) {
          return this.dateFormat(date);
        }

        return format(date, 'D [' + this.t(format(date, 'MMMM')) + '] YYYY');
      },

      formatHumanShort(date) {
        return format(date, 'D [' + this.t('abr.' + format(date, 'MMM')) + ']');
      },

      formatMonth(month, year) {
        return format(new Date(year, month), '[' + this.t(format(new Date(year, month), 'MMMM')) + '] YYYY');
      },

      format(date, formatDate = 'YYYY-MM-DD') {
        return format(date, formatDate);
      },

      debouncePeople: debounce(function () {
        this.debouncing = false;
      }, 750),

      checkClickOverlay(e) {
        if (e.target.classList.contains('open')) {
          this.open = null;
        }
      },

      checkPeopleRestriction() {
        let count = 0;

        Object.keys(this.selected.people).forEach((id) => {
          let total = parseInt(this.selected.people[id]);

          if (total < 0) {
            this.selected.people[id] = 0;
          }

          if (this.maximumPeople <= 0) {
            return;
          }

          const person = this.people.find((person) => person.id == id)

          if (person && person.max && total > person.max) {
            this.selected.people[id] = (count === 0 ? (person.max || total) : 0);
          }

          if (total > this.maximumPeople) {
            this.selected.people[id] = (count === 0 ? (this.maximumPeople || total) : 0);
          }

          if (count > this.maximumPeople) {
            this.selected.people[id] = 0;
          }

          const previousCount = count;

          count += total;

          if (count > this.maximumPeople) {
            this.selected.people[id] = this.maximumPeople - (count - total);
          }
        });

        if ((count <= 0 || count < this.minimumPeople) && this.people.length > 0) {
          this.selected.people[this.people[0].id] = this.minimumPeople;
        }
      },

      updateDisabledDepartureDates() {
        const arrivalDate = this.availability.find((date) => (
          isEqual(date.date, this.selected.arrival)
        ));

        const notDepartures = this.availability.filter((date) => !date.departure).map((date) => (
          date.date
        ));

        if (!arrivalDate || !arrivalDate.periods || arrivalDate.periods.length === 0) {
          return;
        }

        if (!arrivalDate) {
          return this.dates.departure.disabled.dates = notDepartures;
        }

        const nights = arrivalDate.periods.map(o => (
          o.nights
        )).sort().reverse();

        const last = this.dates.departure.disabled.from = addDays(arrivalDate.date, nights[0]);

        eachDay(arrivalDate.date, last).forEach(date => {
          const index = nights.indexOf(diffDays(date, arrivalDate.date));

          if (index === -1) {
            notDepartures.push(this.format(date));
          }
        });

        this.dates.departure.disabled.dates = notDepartures;
      },

      updatePeriod() {
        const arrivalDate = this.availability.find((date) => (
          isEqual(date.date, this.selected.arrival)
        ));

        const departureDate = this.availability.find((date) => (
          isEqual(date.date, this.selected.departure)
        ));

        if (this.options.periods && this.periods.length > 0) {
          const diff = diffDays(this.selected.departure, this.selected.arrival);

          const period = this.periods.find((item) => (
            item.nights == diff
          ))

          if (period) {
            this.selected.period = period.id;
          }
        }

        if (!arrivalDate || !departureDate) {
          return;
        }

        if (!arrivalDate.periods || arrivalDate.periods.length === 0) {
          return;
        }

        const period = arrivalDate.periods.find((item) => (
          item.nights == diffDays(departureDate.date, arrivalDate.date)
        ))

        this.periodDate = period;
      },

      update() {
        const session = Object.assign({}, this.selected);

        if (!this.settings.general) {
          Object.keys(session).forEach((key) => (
            session[key] === null ? session[key] = this.previous[key] : null
          ));
        }

        if (session.arrival && session.departure && isAfter(session.arrival, session.departure)) {
          session.departure = null;
        }

        this.$storage.save('selected', session);

        const filters = {...session.filters, ...session.searchFilters};

        this.$emit(
          'update', {...session, ...{filters}}
        );

        bus.$emit('search.selected', {...session, ...{filters}});
      },

      silentUpdate() {
        this.$emit(
          'silentUpdate', this.$storage.save('selectedSilent', this.selected)
        );

        this.$storage.save('selected', this.selected);

        const filters = {...this.selected.filters, ...this.selected.searchFilters};

        bus.$emit('search.selected', {...this.selected, ...{filters}});
      },

      submit() {
        if (this.$apollo.queries.availability.loading) {
          return;
        }

        this.update();

        this.$emit('close', true);

        if (this.url) {
          const link = document.createElement('a');
          link.href = this.url;

          if (link.host !== window.location.host) {
            const query = {
              h_category: this.selected.category,
              h_accommodation: this.selected.accommodation,
              h_arrival: this.format(this.selected.arrival),
              h_departure: this.format(this.selected.departure),
            };

            Object.keys(this.selected.people).forEach((id) => {
              query['h_people[' + id + ']'] = this.selected.people[id]
            });

            this.redirectOverlay = true;

            setTimeout(() => {
              window.location.href = this.url + '?' + getQueryString(query);
            }, 1000 * 2)
          } else {
            window.location.href = this.url;
          }
        } else {
          this.expanded = false;
          this.open = null;
        }
      },

      toggle(box) {
        this.open = this.open === box ? null : box;
      },

      getProbableArrivalDate(dates, findClosest = false, firstDate) {
        firstDate = firstDate || new Date;

        if (findClosest) {
          firstDate = closestTo(
            firstDate, dates.filter(date => date.arrival && !date.occupied).map(date => (date.date))
          )
        }

        const arrival = dates.find((date) => (
          date.arrival && isAfter(date.date, subDays(firstDate, 1))
        ));

        if (arrival) {
          const minimumDays = Math.max(arrival.minimum_duration || 0, this.accommodation.minimum_days || 0, this.category.minimum_days || 0);

          const departure = dates.find((date) => (
            date.departure && isAfter(date.date, addDays(arrival.date, (minimumDays >= 1 ? (minimumDays - 1) : 0)))
          ));

          if (!departure) {
            return arrival;
          }

          if (this.periodHasOccupipedDays(dates, arrival.date, departure.date)) {
            return this.getProbableArrivalDate(dates, false, addDays(firstDate, 1));
          }
        }

        return arrival;
      },

      periodHasOccupipedDays(dates, arrival, departure) {
        if (diffDays(departure, arrival) == 1) {
          return false;
        }

        const occupiedDate = dates.find((date) => (
          date.occupied &&
          (isEqual(date.date, arrival) || isAfter(date.date, arrival)) &&
          (isBefore(date.date, departure))
        ));

        return !!occupiedDate;
      },

      select(item, value) {
        this.$set(this.selected, item, value);

        if (item === 'category') {
          this.$set(this.selected, 'searchCategory', value);
        }
      },
    },
  };
</script>
