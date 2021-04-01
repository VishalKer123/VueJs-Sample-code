<template>
    <div class="harbor-widget harbor-widget-overview" :class="{['h-widget-' + theme]: true, 'h-widget-initial': initial, 'h-widget-loading': $apollo.queries.overview.loading, 'h-widget-no-results': filtered.length === 0, 'h-widget-light': light, 'harbor-widget-overview-filters': settings.filter, 'harbor-widget-overview-overlay': overlay, 'overflow-hidden': mapShown}" ref="container">
        <search v-if="!light" v-show="show.search" :category-id="categoryId" :filtered="filteredCategories" :expandable="expandable" :config="searchConfig" :override-people="overridePeople" :load="load" :label="t('Wijzig uw gegevens')" :filters="filters" :pre-fill="preFill" @update="listenToSearch" ref="search" :searchFilters="searchFilters" :custom-search-filters-label="customSearchFiltersLabel" :override-remember-filters="overrideRememberFilters" :additional-information="additionalInformation" :bar-theme="barTheme" />

        <div class="h-accommodations-details" v-if="filtered && filtered.length && !light && !overlay">
            <template v-if="settings.overviewCategories">
                <div class="h-found">
                    <span class="h-value">{{ filteredOverviewCategories.length }}</span> <span class="h-label">{{ t(filteredOverviewCategories.length === 1 ? 'accommodatie' : 'accommodaties') }}</span>
                </div>
            </template>
            <template v-else>
                <div class="h-found">
                    <span class="h-value">{{ filtered.length }}</span> <span class="h-label">{{ t(filtered.length === 1 ? 'accommodatie' : 'accommodaties') }}</span>
                </div>
            </template>

            <div class="h-details">
                <span class="h-detail-people">{{ countedPeople }} {{ t(countedPeople !== 1 ? 'personen' : 'persoon') }}</span>
                <span class="h-detail-period" v-if="search.departure"><span class="h-arrival">{{ formatHuman(search.arrival) }}</span> <span class="h-departure">{{ formatHuman(search.departure) }}</span></span>
                <span class="h-detail-period" v-else-if="search.period && period.id"><span class="h-arrival">{{ formatHuman(search.arrival) }}</span> <span class="h-period">{{ period.title }}</span></span>
            </div>
        </div>

        <div class="h-accommodations-container">
            <div class="h-accommodations-filters" :class="{ 'h-accommodations-filters-open': show.filters }" v-if="settings.filter || settings.filters">
                <a v-if="settings.filter && settings.filter.type === 2" class="h-filters-toggle" :class="{ 'h-toggle-open': show.filters }" @click="$event.preventDefault(); show.filters = !show.filters"></a>

                <div class="h-accommodations-filters-container">
                  <template v-if="settings.filter && settings.filter.type === 2">
                    <div class="select-input-container" v-for="(value, i) in people" :key="i">
                        <label>{{ value.title }}</label>
                        <div class="h-filter h-input">
                            <div class="h-input-item h-input-filter h-modifiers">
                                <button class="h-modifier h-modifier-minus" :class="{ 'h-disabled': search.people[value.id] === 0 || countedPeople <= 1 }" @click="(countedPeople > 1) ? search.people[value.id]-- : null"></button>
                                <input class="h-input-field" type="text" v-model="search.people[value.id]">
                                <button class="h-modifier h-modifier-plus" :class="{ 'h-disabled': countedPeople === maximumPeople || (value.max ? search.people[value.id] >= value.max : false) }" @click="(countedPeople < maximumPeople && (value.max ? search.people[value.id] < value.max : true)) ? search.people[value.id]++ : null"></button>
                            </div>
                        </div>
                    </div>
                  </template>
                  <button v-if="showFilterCloseBtn" class="h-close" @click="show.filters = false"></button>
                  <div class="h-collapsables h-filter-categories">
                      <template v-for="category in filterCategories">
                          <div :key="category.id" v-if="canShowFilterCategory(category)" class="h-collapsable h-collapsable-filter-category" :class="{ 'h-open': collapsed.indexOf(category.id) !== -1, 'h-or-category': category.or_select }">
                              <div class="h-collapsable-title" @click="toggleCollapsed(category.id)">{{ category.title }}</div>
                              <div class="h-collapsable-body">
                                  <div class="h-filters">
                                      <div class="h-filter h-input" :class="{ 'h-filter-disabled': isFilterDisabled(filter.id, null, category.or_select), ['h-filter-type-' + (['checkbox', 'modifiers', 'select'][filter.type])]: true }" v-for="filter in category.filters" :key="filter.id">
                                          <input type="checkbox" class="h-input-checkbox" :value="true" v-model="search.filters[filter.id]" :id="'h-filter-checkbox-' + filter.id" :disabled="isFilterDisabled(filter.id, null, category.or_select)" v-if="filter.type === 0">
                                          <label class="h-input-label" :for="'h-filter-checkbox-' + filter.id">{{ filter.title }}</label>
                                          <div class="h-input-item h-input-filter h-modifiers" v-if="filter.type === 1">
                                              <button class="h-modifier h-modifier-minus" :class="{ 'h-disabled': search.filters[filter.id] === 0 }" @click="(search.filters[filter.id] > 0) ? search.filters[filter.id]-- : null"></button>
                                              <input class="h-input-field" type="text" v-model="search.filters[filter.id]">
                                              <button class="h-modifier h-modifier-plus" :class="{ 'h-disabled': search.filters[filter.id] >= maximumFilterValue(filter.id) }" @click="search.filters[filter.id] >= maximumFilterValue(filter.id) ? null : search.filters[filter.id]++"></button>
                                          </div>
                                          <div class="h-input-item h-input-filter h-select" v-if="filter.type === 2">
                                              <select v-model="search.filters[filter.id]">
                                                  <option :value="null" :selected="search.filters[filter.id] === null" v-if="!settings.filter || settings.filter.id != filter.id">Selecteer</option>
                                                  <option :selected="search.filters[filter.id] == i" :value="i" v-for="(choice, i) in filter.choices" :key="i">{{ choice }}</option>
                                              </select>
                                          </div>
                                      </div>
                                  </div>
                              </div>
                          </div>
                      </template>
                  </div>

                  <a href="#" class="h-button-reset" @click="resetFilters">Reset filters</a>

                  <a class="h-button h-show-results" href="#" @click="$event.preventDefault(); show.filters = false">
                      {{ t('Toon resultaten') }}
                  </a>
                </div>
            </div>

            <div class="h-accommodations-result">
                <div class="h-accommodations-actions" v-if="!!settings.filter || settings.filters">
                    <a class="h-action-button h-action-filters" v-if="settings.filters" :class="{ active: !show.filters }" href="#" @click="$event.preventDefault(); show.filters = !show.filters">
                        {{ t('Filter uw resultaat') }}
                    </a>
                    <div class="h-action-select h-action-sort" v-if="settings.filters">
                        <select v-model="sort">
                            <option :value="null">{{ t('Sorteer op') }}</option>
                            <option value="name-asc">{{ t('Naam A-Z') }}</option>
                            <option value="name-desc">{{ t('Naam Z-A') }}</option>
                            <option value="price-asc">{{ t('Prijs Laag-Hoog') }}</option>
                            <option value="price-desc">{{ t('Prijs Hoog-Laag') }}</option>
                        </select>
                    </div>

                    <a class="h-action h-action-grid" :class="{ active: !show.map }" href="#" @click="$event.preventDefault(); show.map = false"></a>
                    <a class="h-action h-action-map" :class="{ active: show.map }" href="#" @click="$event.preventDefault(); show.map = true"></a>
                </div>

                <div class="h-accommodations-messages">
                    <div class="h-accommodations-loading" v-show="$apollo.queries.overview.loading">
                        <span class="h-loading"></span> <span class="h-label">{{ t('Accommodaties aan het ophalen') }}</span>
                    </div>

                    <div class="h-accommodations-not-found" v-show="!$apollo.queries.overview.loading && hasBeenSubmitted && (filtered.length === 0 || (settings.overviewCategories && filteredOverviewCategories.filter((item) => item.available !== false).length === 0))">
                        <span class="h-label">{{ t(light && !categoryId ? 'Geen categorie meegegeven' : 'Geen accommodaties gevonden') }}</span>
                    </div>
                </div>

                <div class="h-accommodations-view h-view-list" v-show="!show.map">
                    <div class="h-accommodations" v-if="!settings.overviewCategories && !$apollo.queries.overview.loading">
                        <div class="h-columns">
                            <div class="h-column" v-for="(accommodation, index) in filtered" :key="accommodation.id">
                                <component :is="'accommodation-' + theme" :categoryId="search.category ? parseInt(search.category) : (categoryId ? parseInt(categoryId) : null)" :position="index + 1" :url="overridePandora || settings.url" :people="search.people" :accommodation="accommodation" :show-dates="light || alternatives" :arrival="search.arrival" :departure="search.departure" :period="period || periodDate" :more-info="config.more" :show-person-price="showPersonPrice" :book-now-buttons="accommodationBookNowButtons" :show-information="showInformation" :show-features="showFeatures" @info-height="setInfoHeight" :info-height="infoHeight" :more-info-link-target="moreInfoLinkTarget" :forMap="overlay" :informative="overlay" :book-now-reserve-button="bookNowReserveButton" :book-now-more-info-button="bookNowMoreInfoButton" :book-label="overridePandoraLabel" :expandable-information="expandableInformation" :sub-title-on-top="subTitleOnTop" :alternative-prices="alternativePrices" />
                            </div>
                        </div>
                    </div>

                    <div class="h-accommodations" v-if="settings.overviewCategories && !$apollo.queries.overview.loading">
                        <div class="h-columns">
                            <div class="h-column" v-for="(category, index) in filteredOverviewCategories.filter((item) => item.available !== false)" :key="category.id">
                                <component :is="'accommodation-' + theme" :categoryId="search.category" :position="index + 1" :show-dates="light" :show-price="settings.prices" :url="overridePandora || settings.url" :people="search.people" :category="category" :arrival="search.arrival" :departure="search.departure ? format(search.departure) : departureByPeriod" :period="period || periodDate" @go-out="update" :book-now-buttons="accommodationBookNowButtons" :show-information="showInformation" :show-features="showFeatures" @info-height="setInfoHeight" :info-height="infoHeight" :more-info-link-target="moreInfoLinkTarget" :book-now-reserve-button="bookNowReserveButton" :book-now-more-info-button="bookNowMoreInfoButton" :book-label="overridePandoraLabel" :expandable-information="expandableInformation" :sub-title-on-top="subTitleOnTop" :show-accommodation-options="showAccommodationOptions" />
                            </div>
                        </div>
                    </div>
                </div>

                <div class="h-accommodations-view h-view-map" v-if="show.map">
                    <map-container :all="false" :color="color" :marker="marker" :prices="showMapPrice" :given="filteredOverviewCategories" />
                </div>

                <div class="h-accommodations h-accommodations-not-available" v-if="!$apollo.queries.overview.loading" v-show="notAvailable.length > 0">
                    <div class="h-info">
                      <div class="h-title">{{ t('Alternatieven') }}</div>
                      <div class="h-description">{{ t('Onze alternatieven') }}</div>
                    </div>
                    <div class="h-columns" v-if="!settings.overviewCategories">
                        <div class="h-column" v-for="accommodation in notAvailable" :key="accommodation.id">
                            <component :is="'accommodation-' + theme" :categoryId="search.category" :url="overridePandora || settings.url" :show-dates="true" :people="search.people" :accommodation="accommodation" :button="true" :arrival="search.arrival" :departure="search.departure" :averageMetrics="activeCategory.average_metrics || 0" :book-now-buttons="accommodationBookNowButtons" :show-information="showInformation" :show-features="showFeatures" @info-height="setInfoHeight" :info-height="infoHeight" :more-info-link-target="moreInfoLinkTarget" :book-now-reserve-button="bookNowReserveButton" :book-now-more-info-button="bookNowMoreInfoButton" :book-label="overridePandoraLabel" :expandable-information="expandableInformation" :alternative="expandableInformation" :period="period || periodDate" :sub-title-on-top="subTitleOnTop" :alternative-prices="alternativePrices" />
                        </div>
                    </div>
                    <div class="h-columns" v-else>
                        <div class="h-column" v-for="category in notAvailable" :key="category.id">
                            <component :is="'accommodation-' + theme" :categoryId="search.category" :show-dates="true" :url="overridePandora || settings.url" :people="search.people" :category="category" :arrival="search.arrival" :departure="departureByPeriod" :book-now-buttons="accommodationBookNowButtons" :show-information="showInformation" :show-features="showFeatures" @info-height="setInfoHeight" :info-height="infoHeight" :more-info-link-target="moreInfoLinkTarget" :book-now-reserve-button="bookNowReserveButton" :book-now-more-info-button="bookNowMoreInfoButton" :book-label="overridePandoraLabel" :expandable-information="expandableInformation" :period="period || periodDate" :sub-title-on-top="subTitleOnTop" />
                        </div>
                    </div>
                </div>

                <transition name="slide-fade">
                    <div class="h-notifications" v-if="!$apollo.queries.overview.loading && !$apollo.queries.notifications.loading && show.notification > 0 && notifications.length > 0">
                        <div class="h-notification">
                            <a href="#" class="h-close" @click="$event.preventDefault(); show.notification = 0"></a>
                            <div class="h-title">
                                {{ notifications[0].title }}
                            </div>
                            <div class="h-information" v-html="notifications[0].information"></div>
                            <div class="h-tools">
                                <circle-p :total-steps="notificationSteps" :completed-steps="notificationSteps - show.notification" :diameter="16 * 2" />
                                <a class="h-button h-more-information" :href="notifications[0].url">
                                    {{ t('Bekijk de arrangementen') }}
                                </a>
                            </div>
                        </div>
                    </div>
                </transition>
            </div>
        </div>
    </div>
</template>

<script>
  import multiPriceQuery from '../queries/multiPrice.gql';
  import periodsQuery from '../queries/periods.gql';
  import notificationsQuery from '../queries/notifications.gql';
  import filterCategoriesQuery from '../queries/filterCategories.gql';
  import metaQuery from '../queries/meta.gql';
  import format from 'date-fns/format';
  import addDays from 'date-fns/add_days';

  import { shuffle } from '../plugins/request';
  import gql from 'graphql-tag';

  import bus from '../plugins/bus';
  import {url} from "../plugins/pandora";

  export default {
    props: {
      load: {
        type: Boolean,
        default: true,
      },

      expandable: {
        type: Boolean,
        default: false,
      },

      alternatives: {
        type: Boolean,
        default: false,
      },

      light: {
        type: Boolean,
        default: false,
      },

      showSearch: {
        type: Boolean,
        default: true,
      },

      showPersonPrice: {
        type: Boolean,
        default: false,
      },

      showMapPrice: {
        type: Boolean,
        default: false,
      },

      notification: {
        type: Boolean,
        default: false,
      },

      config: {
        type: Object,
        default() {
          return {
            top: true,
            categories: true,
            arrival: true,
            departure: true,
            filter: void 0,
          };
        },
      },

      offset: {
        type: Number,
        default: 0,
      },

      notificationSteps: {
        type: Number,
        default: 25,
      },

      offsetInitial: {
        type: Boolean,
        default: true,
      },

      categoryId: {
        type: Number,
        default: null,
      },

      filteredCategories: {
        type: Array,
        default: null,
      },

      defaultPeople: {
        type: Object,
        default: null,
      },

      defaultPeopleLight: {
        type: Object,
        default: null,
      },

      overridePeople: {
        type: Object,
        default: null,
      },

      preFill: {
        type: Boolean,
        default: false,
      },

      filters: {
        type: Object,
        default: null,
      },

      theme: {
        type: String,
        default: 'normal',
        validator(value) {
          return ['normal', 'alternative'].indexOf(value) !== -1
        }
      },

      marker: {
        type: String,
        default: '/images/marker.png',
      },

      color: {
        type: String,
        default: '#77873c',
      },

      accommodationBookNowButtons: {
        type: Boolean,
        default: false,
      },

      showFeatures: {
        type: Boolean,
        default: true,
      },

      showInformation: {
        type: Boolean,
        default: false,
      },

      overlay: {
        type: Boolean,
        default: false,
      },

      moreInfoLinkTarget: {
        type: String,
        default: '_blank'
      },

      defaultOrder: {
          type: Boolean,
          default: false
      },

      bookNowReserveButton: {
        type: Boolean,
        default: true
      },

      bookNowMoreInfoButton: {
        type: Boolean,
        default: true
      },

      searchFilters: {
        type: Object,
        default: null
      },

      customSearchFiltersLabel: {
        type: String,
        default: null
      },

      overridePandoraUrl: {
        type: String,
        default: null
      },

      overridePandoraLabel: {
        type: String,
        default: null
      },

      overrideRememberFilters: {
        type: Boolean,
        default: false
      },

      expandableInformation: {
        type: Boolean,
        default: false
      },

      additionalInformation: {
        type: Boolean,
        default: true
      },

      barTheme: {
        type: String,
        default: 'default',
        validator(value) {
          return ['default', 'white'].indexOf(value) !== -1
        }
      },

      subTitleOnTop: {
        type: Boolean,
        default: false
      },
      
      initialLight: {
        type: Boolean,
        default: false
      },

      alternativePrices: {
        type: Boolean,
        default: false
      },
      
      showFilterCloseBtn: {
        type: Boolean,
        default: false,
      },
      
      showAccommodationOptions: {
        type: Boolean,
        default: false
      },
    },

    components: {
      accommodationNormal: require('./Accommodation'),
      accommodationAlternative: require('./AccommodationAlt'),
      circleP: require('./Circle'),
      mapContainer: require('./MapCategories'),
    },

    mounted() {
      this.$root.$on('mapShown', () => {
        this.mapShown = true;
      });

      this.$root.$on('mapHidden', () => {
        this.mapShown = false;
      });
    },

    apollo: {
      settings: {
        query: metaQuery,
      },

      categories: {
        query: metaQuery,
      },

      people: {
        query: metaQuery,

        skip() {
          return !this.search.people;
        },

        result({ data }) {
          if (!(this.light || this.considerInitialLight)) {
            return;
          }

          const people = {};
          const hasDefault = data.people.find((item) => (
            item.default > 0
          ));

          data.people.forEach((item, i) => (
            people[item.id] = this.search.people[item.id] === void 0 ? (i === 0 && !hasDefault ? 2 : ((this.defaultPeople && this.defaultPeople[item.id]) || item.default || 0)) : this.search.people[item.id]
          ));

          this.$set(this.search, 'people', people);
        },
      },

      filterCategories: {
        query: filterCategoriesQuery,

        skip() {
          if (!this.settings.filter && !this.settings.filters) {
            return true;
          }

          return false;
        },

        result({ data }) {
          const filters = {};

          data.filterCategories.forEach((category, i) => {
            if (category.open) {
              this.collapsed.push(category.id);
            }

            category.filters.forEach((item) => (
              filters[item.id] = this.search.filters[item.id] === void 0 ? (item.type === 1 ? (item.default || 0) : (item.type === 2 ? null : false)) : this.search.filters[item.id]
            ))
          });

          this.$set(this.search, 'filters', filters);
        },
      },

      periods: {
        query: periodsQuery,

        variables() {
          return {
            category: this.search.category,
            accommodation: null,
          };
        },

        skip() {
          if (!this.settings.periods || !this.search.period) {
            return true;
          }

          if (this.$apollo.queries.settings.loading) {
            return true;
          }

          return false;
        },
      },

      notifications: {
        query: notificationsQuery,

        variables() {
          return {
            category: this.search.category,
            arrival: this.format(this.search.arrival),
          };
        },

        skip() {
          if (!this.notification || !this.search.category) {
            return true;
          }

          if (this.$apollo.queries.settings.loading) {
            return true;
          }

          return false;
        },

        result() {
          if (!this.$apollo.queries.overview.loading) {
            this.showNotifications();
          }
        }
      },

      overview: {
        query() {
          if (this.settings.overviewCategories) {
            return require('../queries/overviewCategories.gql');
          }

          return require('../queries/overview.gql');
        },

        context: {
          batching: false,
        },

        fetchPolicy: 'cache-and-network',

        variables() {
          const people = [];

          Object.keys(this.search.people).forEach((id) => {
            people.push({
              id,
              value: this.search.people[id],
            });
          });

          if (this.light || this.considerInitialLight) {
            return {
              category: (this.categoryId ? this.categoryId : (this.search.category || null)),
              people,
              light: true,
            };
          }

          return {
            category: this.search.category,
            arrival: this.format(this.search.arrival),
            departure: this.search.departure ? this.format(this.search.departure) : null,
            period: this.search.period,
            people: this.settings.filter && this.settings.filter.type === 2 ? null : people,
            alternatives: this.alternatives,
          };
        },

        skip() {
          if ((this.light || this.considerInitialLight) && this.search.people) {
            return this.$apollo.queries.settings.loading;
          }

          if (this.settings.periods && !this.search.period) {
            return true;
          }

          if (this.settings.filter && this.search.people) {
            return false;
          }

          if (!this.settings.periods && !this.search.departure) {
            return true;
          }

          return !this.search.arrival;
        },

        manual: true,

        result({ data, loading }) {
          if (!loading) {
            this.hasBeenSubmitted = true;

            if (data.overviewCategories) {
              this.overview = data.overviewCategories.map((category) => {
                const item =  {
                  ...category,
                  accommodations: category.accommodations.map((accommodation) => ({
                    ...accommodation,
                  })),
                  price: {
                    load: true,
                  },
                };

                if (!!category.accommodations.find((a) => !!a.price)) {
                  const prices = category.accommodations.filter((a) => !!a.price).sort((a, b) => (
                    a.price.total - b.price.total
                  ));

                  item.price = {
                    ...prices[0].price,
                    accommodation: prices[0].id,
                  };
                }

                return item;
              });
            } else if (this.light || this.considerInitialLight) {
              this.overview = data.overview.map((accommodation) => (
                {
                  ...accommodation,
                  available: true,
                  price: {
                    load: this.settings.prices,
                  },
                }
              ));
            } else if (this.alternatives) {
              this.overview = data.overview.map((accommodation) => (
                {
                  ...accommodation,
                  price: {
                    load: this.settings.prices,
                  },
                }
              ));
            } else if (this.settings.asyncPrice) {
              this.overview = data.overview.map((accommodation) => (
                {
                  ...accommodation,
                  price: {
                    load: true,
                  },
                }
              ));
            } else {
              this.overview = data.overview;
            }
            if (this.overview.length) {
              this.overview.forEach((item, key) => {
                let accommodation = this.settings.overviewCategories ? item.accommodations[0] : item;
                let category = this.settings.overviewCategories ? item : (accommodation.categories.find((value) => value.id === this.search.category)) || accommodation.categories.find((value) => value.id === this.categoryId);

                this.$ga.ecommerce.addImpression({
                  'name': accommodation.title,
                  'id': accommodation.provider.key,
                  'price': accommodation.price.total,
                  'category': category ? category.title : null,
                  'list': 'Overview',
                  'position': key + 1
                });
              });

              this.$ga.page();
            }

            if (!this.$apollo.queries.notifications.loading) {
              this.showNotifications();
            }
          }
        },
      },
    },

    data() {
      return {
        settings: {
          periods: false,
          overviewCategories: false,
          filter: null,
        },

        hasBeenSubmitted: false,
        initial: true,
        mapShown: false,

        show: {
          search: this.showSearch,
          filters: false,
          map: false,
          open: false,
          notification: 0,
        },

        tick: {
          notification: null,
        },

        search: {
          category: null,
          arrival: null,
          departure: null,
          period: null,
          people: null,
          filters: {},
        },

        sort: null,

        periodDate: null,

        overview: [],

        collapsed: [],

        periods: [],

        filterCategories: [],

        notifications: [],

        infoHeight: null
      };
    },

    watch: {
      overview() {
        const ids = this.accommodationIds();

        const people = [];

        Object.keys(this.search.people).forEach((id) => {
          people.push({
            id,
            value: this.search.people[id],
          });
        });

        const selected = {
          ...this.$storage.retrieve('selected'),
        };

        if(this.light){
          var today = new Date();
          var jsonItem;
          for (jsonItem in selected) {
              if(jsonItem == 'arrival'){
                  selected[jsonItem] = this.format(today);
              }
          }

          this.$storage.save('selected', selected);
          this.$storage.save('selectedSilent', selected);
        }

        const length = 3;

        if (ids.length > 0 && this.settings.prices) {
          for (let i = 0; i < ids.length; i += length) {
            const accommodations = ids.slice(i, i + length);

            this.$apollo.query({
              query: multiPriceQuery,

              variables: {
                accommodations,
                arrival: this.light || this.considerInitialLight ? null : this.format(this.search.arrival),
                departure: this.light || this.considerInitialLight ? null : (this.search.departure ? this.format(this.search.departure) : null),
                period: this.light || this.considerInitialLight ? null : this.search.period,
                people,
                light: this.light || this.considerInitialLight,
              },

              context: {
                batching: false,
              },
            }).then(({ data }) => {
              data.prices.forEach((price) => {
                const accommodation = this.overview.find((item) => (
                  this.settings.overviewCategories ? (!!item.accommodations.find((a) => a.id == price.accommodation) && !item.price.total) : (item.id === price.accommodation && !item.price.total)
                ));

                if (accommodation) {
                  this.$set(accommodation, 'price', price);
                }
              });

              this.overview.filter((item) => (
                item.price && item.price.load
              )).filter((item) => {
                const list = [
                  ...item.accommodations.map(a => a.id),
                  item.id,
                ];

                return list.filter(i => accommodations.indexOf(i) !== -1).length > 0;
              }).forEach((item) => {
                this.$set(item.price, 'load', false);
              });
                // When multiPrice query is called
                if(this.overview){
                    this.overview.forEach((accommodation, key) => {
                        var cat = accommodation.categories.find((value) => { return value.id == this.categoryId })
                        this.$ga.ecommerce.addImpression({
                            'name': accommodation.title,
                            'id': accommodation.provider.key,
                            'price': accommodation.price.total,
                            'category': cat ? cat.title : null,
                            'list': 'Overview',
                            'position': key + 1
                        });
                        this.$ga.page();
                    });
                }
            });
          }
        }

        /**
         * The unavailable prices are loaded here.
         */
        const unavailableIds = this.accommodationIdsNotAvailable();

        for (let i = 0; i < unavailableIds.length; i += length) {
          const accommodations = unavailableIds.slice(i, i + length);

          this.$apollo.query({
            query: multiPriceQuery,

            variables: {
              accommodations,
              arrival: this.format(this.search.arrival),
              departure: (this.search.departure ? this.format(this.search.departure) : this.departureByPeriod),
              period: this.search.period,
              people,
              light: true,
              single: true,
            },

            context: {
              batching: false,
            },
          }).then(({ data }) => {
            this.overview.filter((item) => (
              item.price && item.price.load && !item.available
            )).filter((item) => (
              accommodations.indexOf(this.settings.overviewCategories ? item.accommodations[0].id : item.id) !== -1
            )).forEach((item) => {
              this.$set(item.price, 'load', false);
            });

            data.prices.forEach((price) => {
              const accommodation = this.overview.find((item) => (
                this.settings.overviewCategories ? !!item.accommodations.find((a) => a.id == price.accommodation) : item.id === price.accommodation
              ));

              this.$set(accommodation, 'price', price);
            });
          });
        }
      },

      'search.filters': {
        deep: true,
        handler(data) {
          if (this.$refs.search === void 0) {
            return;
          }

          const syncable = Object.keys(this.$refs.search.selected.filters);

          syncable.forEach((filter) => (
            data[filter] !== void 0 ? this.$refs.search.selected.filters[filter] = data[filter] : null
          ));

          if (this.settings.filter) {
            const filtered = this.filterCategories.filter((item) => (
              item.show && item.show.filter == this.settings.filter.id && item.show.value != data[this.settings.filter.id]
            )).map((item) => item.filters);

            [].concat(...filtered).forEach((item) => {
              this.search.filters[item.id] = (item.type === 1 ? (item.default || 0) : (item.type === 2 ? null : false))
            });
          }
        },
      },
    },

    computed: {
      countedPeople() {
        if (!this.search.people) {
          return 0;
        }

        return Object.values(this.search.people).reduce((a, b) => parseInt(a) + parseInt(b), 0);
      },

      departureByPeriod() {
        const period = this.periods.find((period) => (
          period.id === parseInt(this.search.period)
        ));

        if (!period) {
          return null;
        }

        return this.format(addDays(this.search.arrival, period.nights));
      },

      period() {
        const period = this.periods.find((period) => (
          parseInt(this.search.period) === period.id
        ));

        if (!period) {
          return null;
        }

        return period;
      },

      activeCategory() {
        const category = (this.categories || []).find((category) => (
          this.search.category === category.id
        ));

        if (!category) {
          return {};
        }

        return category;
      },

      filterList() {
        const filters = this.filterCategories.map((category) => (
          category.filters.map((filter) => ({
            ...filter,
            or_select: category.or_select,
            category: category.id,
          }))
        ));

        return [].concat(...filters);
      },

      filtered() {
        let overview = this.overview;

        if (!this.settings.filter) {
          return this.overview.filter((item) => !this.alternatives || (this.alternatives && item.available));
        }

        const filters = Object.keys(this.search.filters).filter((id) => {
          const info = this.filterList.find((itemFilter) => (
            itemFilter.id == id
          ));

          if (info && info.type == 2) {
            return this.search.filters[id] !== null
          }

          return this.search.filters[id];
        });

        if (this.settings.overviewCategories && this.settings.filter) {
          overview = this.filteredOverviewCategories.flatMap((item) => (
            item.accommodations
          ));

          const listOfIds = overview.map((item) => item.id);

          return [...overview].sort((a, b) => {
            if (a.maximum_people < b.maximum_people) {
              return -1;
            }

            if (a.maximum_people > b.maximum_people) {
              return 1;
            }

            return (listOfIds.indexOf(a.id) - listOfIds.indexOf(b.id));
          });
        }

        const filtered = overview.filter((item) => {
          let available = true;

          filters.forEach((filter) => {
            const found = item.filters.find((itemFilter) => (
              itemFilter.filter == filter
            ));

            const info = this.filterList.find((itemFilter) => (
              itemFilter.id == filter
            ));

            if (!found || !info) {
              available = false;

              if (info && info.type == 1) {
                available = true;
              }

              return;
            }

            if (info.type == 1 && this.search.filters[filter] > found.value) {
              available = false;

              return;
            }

            if (info.type == 2 && this.search.filters[filter] != found.value) {
              available = false;

              return;
            }
          });

          return available;
        });

        if (this.defaultOrder) {
            return [...filtered];
        }

        let listOfIds = this.$storage.retrieveSession('overview.shuffle');

        if (!listOfIds || listOfIds.length < 1) {
          listOfIds = shuffle(overview.map((item) => item.id));

          this.$storage.saveSession('overview.shuffle', listOfIds);
        }

        return [...filtered].sort((a, b) => {
          if (a.maximum_people < b.maximum_people) {
            return -1;
          }

          if (a.maximum_people > b.maximum_people) {
            return 1;
          }

          return (listOfIds.indexOf(a.id) - listOfIds.indexOf(b.id));
        });
      },

      filteredOverviewCategories() {
        if (!this.settings.filter && !this.settings.filters) {
          return this.filtered;
        }

        const filters = Object.keys(this.search.filters).filter((id) => {
          const info = this.filterList.find((itemFilter) => (
            itemFilter.id == id
          ));

          if (info && info.type == 2) {
            return this.search.filters[id] !== null
          }

          return this.search.filters[id];
        });

        const infoFilters = filters.map((filter) => (
          this.filterList.find((itemFilter) => (
            itemFilter.id == filter
          ))
        )).filter((item) => !!item);

        const filtered = this.overview.map((item) => {
          const accommodations = item.accommodations.map((accommodation) => ({ ...accommodation, filters: [...accommodation.filters, ...item.filters] })).filter((accommodation) => {
            let available = true;
            let availableSelect = true;
            let hasFoundOr = [];

            infoFilters.forEach((filter) => {
              const found = accommodation.filters.find((itemFilter) => (
                itemFilter.filter == filter.id
              ));

              if (!found) {
                available = false;

                if (filter.type == 1) {
                  available = true;
                }

                return;
              }

              if (filter.type == 0 && filter.or_select) {
                hasFoundOr.push(filter.category);
              }

              if (filter.type == 1 && this.search.filters[filter.id] > found.value) {
                available = false;

                return;
              }

              if (filter.type == 2 && this.search.filters[filter.id] != found.value) {
                available = false;
                availableSelect = false;

                return;
              }
            });

            const foundFilter = infoFilters.find((filter) => {
              return hasFoundOr.indexOf(filter.category) !== -1;
            });

            if (foundFilter) {
              available = true;
            }

            if (this.countedPeople > accommodation.maximum_people) {
              available = false;
            }

            if (!availableSelect) {
              available = false;
            }

            return available;
          });

          let price = item.price;

          if (!!accommodations.find((a) => !!a.price)) {
              const prices = accommodations.filter((a) => !!a.price).sort((a, b) => (
                a.price.total - b.price.total
              ));

              price = {
                ...prices[0].price,
                accommodation: prices[0].id,
              };
          }

          return {
            ...item,
            accommodations,
            price
          };
        });

        const filteredAccommodations = filtered.filter((item) => (
          item.accommodations.length > 0
        ));

        if (this.settings.filters) {
          return filteredAccommodations.sort((a, b) => {
            switch (this.sort) {
              case 'price-desc':
                return a.price.total < b.price.total ? 1 : -1;
              case 'price-asc':
                return a.price.total > b.price.total ? 1 : -1;
              case 'name-desc':
                return a.title < b.title ? 1 : -1;
              case 'name-asc':
              default:
                return a.title > b.title ? 1 : -1;
            }
          });
        }

        return filteredAccommodations;
      },

      notAvailable() {
        if (!this.alternatives) {
          return [];
        }

        if (this.settings.overviewCategories && this.settings.filters) {
          return this.filteredOverviewCategories.filter((item) => item.available === false);
        }

        return this.overview.filter((item) => !item.available);
      },

      searchConfig() {
        return {
          ...this.config,
          arrival: !this.light && this.config.arrival,
          departure: !this.light && this.config.departure,
        };
      },

      maximumPeople() {
        let categories = this.overview;

        if (!this.settings.filter) {
          return 0;
        }

        if (this.settings.filter && this.settings.filter.type === 2) {
          categories = categories.map((category) => {
            category = {
              ...category,
              accommodations: category.accommodations.filter((accommodation) => {
                const found = accommodation.filters.find((filter) => (
                  this.settings.filter.id === filter.filter && filter.value == this.search.filters[this.settings.filter.id]
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

        if (max === -Infinity) {
          return 0;
        }

        if (this.countedPeople > max) {
          this.search.people[this.people[0].id] = max;
        }

        if (max <= 0) {
          return 0;
        }

        return max;
      },

      accommodations() {
        let filtered = this.filtered;

        if (this.settings.overviewCategories) {
          filtered = this.filteredOverviewCategories;
        }

        return filtered.filter(iteration => !!iteration.accommodations)
          .map(iteration => iteration.accommodations).flat(1);
      },

      overridePandora() {
        if (!this.overridePandoraUrl) {
          return;
        }

        if (this.settings.urlQuery) {
          return this.overridePandoraUrl + '?' + this.settings.urlQuery;
        }

        return this.overridePandoraUrl;
      },

      considerInitialLight() {
        if (!this.initialLight) {
          return false;
        }

        if (!this.settings.periods && !this.search.departure) {
            return true;
        }

        return !this.search.arrival;
      }
    },

    created() {
      if (this.light) {
        let session = {
          ...this.$storage.retrieve('selected'),
        };

        this.search.people = this.defaultPeople ? {} : (session.people ? session.people : {});

        bus.$once('search.selected', ({category, queryFilters}) => {
          if (category) {
            this.search.category = parseInt(category);
          }

          if (queryFilters && typeof queryFilters === 'object') {
            Object.keys(queryFilters).forEach((item) => (
              this.search.filters[item] = queryFilters[item]
            ));
          }
        });
      }

      if (this.filters) {
        Object.keys(this.filters).forEach((item) => (
          this.search.filters[item] = this.filters[item]
        ));
      }
    },

    methods: {
      formatHuman(date) {
        return format(date, 'D [' + this.t(format(date, 'MMMM')) + ']');
      },

      format(date, formatDate = 'YYYY-MM-DD') {
        return format(date, formatDate);
      },

      accommodationIds() {
        return this.overview.filter((item) => (
          item.price && item.price.load && item.available
        )).map((item) => {
          if (!item.accommodations) {
            return item.id;
          }

          const list = [...item.accommodations].sort((a, b) => {
            if (a.maximum_people < b.maximum_people) {
              return -1;
            }

            if (a.maximum_people > b.maximum_people) {
              return 1;
            }
          });

          const accommodations = list.filter((item) => (
            item.maximum_people >= this.countedPeople
          ));

          if (accommodations.length) {
              return accommodations.map(a => a.id);
          }

          return item.accommodations.map(a => a.id);
        }).flat(1);
      },

      accommodationIdsNotAvailable() {
        return this.overview.filter((item) => (
          item.price && item.price.load && item.available === false
        )).map((item) => {
          if (!item.accommodations) {
            return item.id;
          }

          const list = [...item.accommodations].sort((a, b) => {
            if (a.maximum_people < b.maximum_people) {
              return -1;
            }

            if (a.maximum_people > b.maximum_people) {
              return 1;
            }
          });

          const accommodations = list.filter((item) => (
            item.maximum_people >= this.countedPeople
          ));

          if (accommodations.length) {
              return accommodations.map(a => a.id);
          }

          return item.accommodations.map(a => a.id);
        }).flat(1);
      },

      isFilterDisabled(id, value = null, isOr = false) {
        if (isOr) {
          return false;
        }

        let filtered = this.filtered;

        if (this.settings.overviewCategories) {
          filtered = this.filteredOverviewCategories;
        }

        let disabled = !filtered.find((item) => (
          !!item.filters.find((filter) => (
            filter.filter == id && (value !== null ? filter.value == value : true)
          ))
        ));

        if (disabled) {
          disabled = !this.accommodations.find((item) => (
            !!item.filters.find((filter) => (
              filter.filter == id && (value !== null ? filter.value == value : true)
            ))
          ));
        }

        return disabled;
      },

      resetFilters(e) {
        e && e.preventDefault();

        const filters = {};

        this.filterCategories.forEach((category) => {
          category.filters.forEach((item) => (
            // filters[item.id] = this.$refs.search.selected.filters[item.id] === void 0 ? (item.type === 1 ? (item.default || 0) : (item.type === 2 ? null : false)) : this.$refs.search.selected.filters[item.id]
            filters[item.id] = this.settings.filter && this.settings.filter.id == item.id ? (this.$refs.search.selected.filters[item.id] || this.settings.filter.default) : (item.type === 1 ? (item.default || 0) : (item.type === 2 ? null : false))
          ))
        })

        this.$set(this.search, 'filters', filters);

        this.update();
      },

      canShowFilterCategory(category) {
        const filters = this.filterList;

        if (!category.visible) {
          return false;
        }

        if (!category.show) {
          return true;
        }

        return this.search.filters[category.show.filter] == category.show.value;
      },

      maximumFilterValue(id) {
        const total = this.filtered.map((item) => {
          const filter = item.filters.find((filter) => (
            filter.filter == id
          ));

          return filter && filter.value ? filter.value : 0;
        });

        return total.reduce((p, v) => (
          p > v ? p : v
        ), 0);
      },

      showNotifications() {
        if (this.notifications.length > 0) {
          this.show.notification = this.notificationSteps;

          this.tick.notification = setInterval(() => {
            this.show.notification--;

            if (this.show.notification < 1) {
              clearInterval(this.tick.notification);
            }
          }, 1000);
        }
      },

      toggleCollapsed(id) {
        if (this.collapsed.indexOf(id) === -1) {
          this.collapsed = [id];
          // this.collapsed.push(id);
        } else {
          this.collapsed = [];
          // this.collapsed.splice(this.collapsed.indexOf(id), 1)
        }
      },

      update(item, e) {
        // e && e.preventDefault();

        const filters = {};

        Object.keys(this.search.filters).forEach((filter) => {
          if (this.search.filters[filter]) {
            filters[filter] = this.search.filters[filter];
          }
        });

        const selected = {
          ...this.$storage.retrieve('selected'),
          filters,
        };

        this.$storage.save('selected', selected);
      },

      listenToSearch({ category, arrival, departure, period, people, filters, queryFilters }) {
        this.search.category = category;
        this.search.arrival = arrival ? this.format(arrival) : arrival;
        this.search.departure = departure ? this.format(departure) : departure;
        this.search.period = period;
        this.search.people = this.settings.filter ? people : Object.assign({}, people);
        this.search.filters = {
          ...this.search.filters,
          ...filters,
          ...queryFilters
        };

        this.$nextTick(() => {
          this.periodDate = this.$refs.search ? (this.$refs.search.periodDate && this.$refs.search.periodDate.id ? this.$refs.search.periodDate : this.$refs.search.period) : null
        })

        if ((!this.initial || this.offsetInitial) && this.offset !== null) {
          window.scrollBy({
            top: this.$refs.container.getBoundingClientRect().top + this.offset,
            behavior: 'smooth',
          });
        }

        this.initial = false;
      },

      setInfoHeight(height) {
        if (!this.infoHeight) {
          this.infoHeight = height;
        }

        if (height > this.infoHeight) {
          this.infoHeight = height;
        }
      }
    },
  };
</script>
