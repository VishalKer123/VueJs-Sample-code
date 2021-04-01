<template>
    <div class="h-accommodation" :class="{ 'h-accommodation-has-packages': item.packages && item.packages.length > 0, 'h-accommodation-has-more-telesto': showAccommodations && item.accommodations && item.accommodations.length >= 6, 'h-more': show.moreDelay, open: show.more, 'h-disabled': (onlySelected && selected !== null && !(selected === item.id)), 'h-info-expandable': expandableInformation, 'h-accommodation-options': showAccommodationOptions }" :style="style" ref="accommodation">
        <div class="h-accommodation-basic" ref="basic">
            <div class="h-accommodation-images">
                <div class="h-accommodation-images-container" ref="images">
                    <div class="h-accommodation-images-inner" :style="{ transform: `translate(${slideOffset}px, 0)`, transition: carousel.drag.dragging ? 'none' : 'transform 0.5s ease 0s' }">
                        <div class="h-accommodation-image" v-for="i in item.images" :key="i">
                            <img :src="i"/>
                        </div>
                    </div>
                </div>

                <div class="h-accommodation-discount" v-if="item.price && item.price.old">
                    <div class="h-discount">
                        {{ Math.ceil((item.price.old - item.price.total) / item.price.old * 100) }}% {{ t('korting') }}
                    </div>
                </div>

                <ol class="h-accommodation-images-indicators" v-if="item.images && item.images.length > 1" v-show="false">
                    <li v-for="(image, i) in item.images.length" @click="carousel.current = i" :class="{ active: i == carousel.current }" :key="i"></li>
                </ol>

                <div class="h-accommodation-images-arrows" v-if="item.images && item.images.length > 1">
                    <a href="#" class="h-arrow-left" :class="{ 'h-disabled': carousel.current === 0 }" @click="$event.preventDefault(); carousel.current > 0 && carousel.current--;"></a>
                    <a href="#" class="h-arrow-right" :class="{ 'h-disabled': carousel.current === item.images.length - 1 }" @click="$event.preventDefault(); carousel.current++; carousel.current > item.images.length - 1 && (carousel.current = item.images.length - 1);"></a>
                </div>
            </div>
            <a class="h-accommodation-title" :href="item.url" @click="viewItem(item, $event, 'viewing')">
                {{ item.title }}
            </a>
            <div class="h-sub-accommodation" v-if="subTitleOnTop && item.accommodations && !showAccommodationOptions">
                {{ subItem && subItem.title }}
            </div>
            <div class="h-accommodation-breadcrumbs" v-if="item.breadcrumbs && item.breadcrumbs.length > 0">
                {{ item.breadcrumbs.join(', ') }}
            </div>
            <div class="h-accommodation-features" v-if="item.features && item.features.length > 0 && showFeatures">
                <ul>
                    <li v-for="(feature, i) in filterFeatures(item.features)" :key="i" :class="{ 'h-has-tooltip': true }" :title="feature.value + ' ' + feature.title">
                        <template v-if="typeof feature === 'object' && (feature.title || feature.icon)">
                            <i class="fas" :class="['fa-' + feature.icon]"></i>
                            <span class="h-value">{{ feature.value }}</span>
                            <span class="h-label">{{ feature.title }}</span>
                        </template>
                        <span v-else-if="typeof feature === 'string'">{{ feature }}</span>
                    </li>
                    <li v-if="item.review">
                        <div :class="['h-review-provider', item.review.provider]"></div>
                        <span class="h-value">{{ item.review.rating }}</span>
                    </li>
                    <!-- <li class="h-feature-more" v-if="item.features && item.features.length > 0">
                        <a href="#" class="h-button-inline h-button-more" @click="$event.preventDefault(); show.more = true; show.moreDelay = true; show.packages = false; show.price = false"></a>
                    </li> -->
                </ul>
            </div>
            <div ref="accommodationInformation" class="h-accommodation-information" :class="{ 'h-more-info': expandableInformation && show.moreInfo, 'h-truncated-info': truncated.information }" v-if="item.information && showInformation" :style="{minHeight: infoHeight + 'px'}">
                <span v-html="item.information"></span>
                <div class="h-tooltip" v-if="expandableInformation && truncated.information">
                    <a @click.prevent="toggleShowInfo" :title="item.information" class="h-information-more"></a>
                </div>
            </div>
            <div class="h-accommodation-tags" v-if="item.tags && item.tags.length > 0 && showTags">
                <ul>
                    <li v-for="(tag, i) in item.tags" :key="i">
                        {{ tag }}
                    </li>
                </ul>
            </div>
            <div class="h-accommodation-details">
                <template v-if="showAccommodationOptions">
                  <div class="h-accommodations">
                    <div class="h-accommodation-item" v-for="(accommodationItem, i) in category.accommodations" :key="accommodationItem.id" ref="accommodationOptions" :class="{'h-selected': selection.accommodation === accommodationItem.id}">
                      <div class="h-accmmodation-name">
                        <input type="radio" :name="category.title + '-accommodation'" :id="'accommodation-' + accommodationItem.id" :value="accommodationItem.id" v-model="selection.accommodation" />
                        <label class="h-accommodation-label" :for="'accommodation-' + accommodationItem.id">
                          <span class="h-title-truncated">{{ accommodationItem.title }}</span>

                          <div class="h-tooltip" v-if="truncated.accommodations.indexOf('a' + i) !== -1">
                              <a href="#" @click="$event.preventDefault()" :title="accommodationItem.title" class="h-price-more"></a>
                          </div>
                        </label>
                      </div>
                      <div class="h-accommodation-price">
                          <div class="h-price" v-if="accommodationItem.price && accommodationItem.price.base">
                              <span class="h-value-old" v-if="accommodationItem.price.old">€ {{ priceFormat(accommodationItem.price.old) }}</span>
                              <div class="h-price-line">
                                  <span class="h-value">€ {{ priceFormat(accommodationItem.price.total) }}</span>
                                  <price :categoryId="categoryId ? parseInt(categoryId) : null" :title="item.title" :second-title="accommodationItem.title" :price="accommodationItem.price" :details="{ category, accommodationItem, people, arrival: (accommodationItem.price && accommodationItem.price.arrival) || arrival, departure: (accommodationItem.price && accommodationItem.price.departure) || departure }" :url="pandora" ref="price" :new-structure="priceNewStructure" />
                              </div>
                          </div>
                          <div v-else class="h-price">
                              <span class="h-value">{{ t('Geen prijs') }}</span>
                          </div>
                        <div class="h-label-help h-label-deposit" v-if="accommodationItem.price.articles && accommodationItem.price.articles.length === 1 && depositBelowPrice">
                            <span class="h-label">{{ t('Exclusief') }}</span> <span class="h-value">€ {{ priceFormat(accommodationItem.price.articles[0].price) }}</span> <span class="h-label-append">{{ t('borg') }}</span>
                        </div>
                      </div>
                    </div>
                  </div>
                </template>
                <template v-else-if="item.price && showPrice">
                    <div class="h-accommodation-price-label" :class="{ 'h-label-show-dates': showDates }">
                        <span class="h-price-accommodation" v-if="!subTitleOnTop && item.accommodations">{{ subItem && subItem.title }}</span>

                        <span class="h-price-label" v-if="!showDates">{{ t('Uw prijs') }}</span>
                        <div class="h-label-dates" v-if="showDates && item.price">
                            <template v-if="item.price && item.price.load">
                                <span class="h-message">{{ t('Prijs wordt opgehaald') }}</span>
                            </template>
                            <template v-else-if="item.price.arrival">
                                <span class="h-arrival">{{ format(item.price.arrival, 'DD/MM') }}</span> <span class="h-insert">t/m</span> <span class="h-departure">{{ format(item.price.departure, 'DD/MM') }}</span>
                            </template>
                        </div>
                        <div class="h-label-help h-label-person" v-if="item.price.person && showPersonPrice">
                            <span class="h-value">€ {{ priceFormat(item.price.person) }}</span> <span class="h-label-append">{{ t('pp') }}</span>
                        </div>
                        <div class="h-label-help h-label-deposit" v-if="item.price.articles && item.price.articles.length === 1 && !depositBelowPrice">
                            <span class="h-label">{{ t('Exclusief') }}</span> <span class="h-value">€ {{ priceFormat(item.price.articles[0].price) }}</span> <span class="h-label-append">{{ t('borg') }}</span>
                        </div>
                    </div>
                    <div class="h-accommodation-price">
                        <div class="h-loading" v-if="item.price && item.price.load"></div>
                        <template v-else>
                          <div class="h-price" v-if="item.price && item.price.base">
                              <span class="h-value-old" v-if="item.price.old">€ {{ priceFormat(item.price.old) }}</span>
                              <div class="h-price-line">
                                  <span class="h-value">€ {{ priceFormat(item.price.total) }}</span>
                                  <price :categoryId="categoryId ? parseInt(categoryId) : null" :title="item.title" :second-title="subItem && subItem.title" :price="item.price" :details="{ category, accommodation, people, arrival: (item.price && item.price.arrival) || arrival, departure: (item.price && item.price.departure) || departure }" :url="pandora" ref="price" :new-structure="priceNewStructure" />
                              </div>
                          </div>
                          <div v-else class="h-price">
                              <span class="h-value">{{ t('Geen prijs') }}</span>
                          </div>
                        </template>
                        <div class="h-label-help h-label-deposit" v-if="item.price.articles && item.price.articles.length === 1 && depositBelowPrice">
                            <span class="h-label">{{ t('Exclusief') }}</span> <span class="h-value">€ {{ priceFormat(item.price.articles[0].price) }}</span> <span class="h-label-append">{{ t('borg') }}</span>
                        </div>
                    </div>
                </template>
                <template v-else-if="item.available === false && showPrice">
                    <div class="h-accommodation-price-label">
                        <span class="h-label">{{ t('Helaas niet beschikbaar') }}</span>
                    </div>
                    <div class="h-accommodation-price" :class="{ open: show.rhea === item.id }" @click="checkClickOverlay($event)">
                        <a :href="item.url" target="_blank" @click="$event.preventDefault(); show.rhea = item.id; scrollRhea()">{{ t('Bekijk beschikbaarheid') }}</a>

                        <div class="h-accommodation-rhea" v-show="show.rhea === item.id" ref="rhea">
                            <rhea v-if="show.rhea === item.id" :accommodation-id="accommodation ? accommodation.id : null" :category-id="category ? category.id : null" :small="true" :show-price="false" :period-id="period ? period.id : null" :first="item.price ? format(item.price.arrival) : null" />
                        </div>
                    </div>
                </template>
                <div class="h-accommodation-information" v-else-if="!category || (!!category && !showInformation)" v-html="item.information"></div>
            </div>

            <div v-if="accommodation && bookNowButtons" class="h-accommodation-buttons">
                <a v-if="bookNowMoreInfoButton" class="h-button h-button-more-info" :target="accommodation.url ? moreInfoLinkTarget : '_self'" :href="accommodation.url|| 'javascript:void(0);'" :onclick="getTrackEvent('website') || 'ga !== void 0 ? ga(\'send\', \'event\', \'' + accommodation.title + '\', \'card\', \'uitgaand bezoek website button\') : null'">
                    {{ t('Meer info') }}
                </a>
                <a v-if="bookNowReserveButton" class="h-button h-button-book-now" :target="accommodation.booking_url ? '_blank' : '_self'" :href="accommodation.booking_url|| 'javascript:void(0);'" @click="getTrackEvent('booking', accommodation) || 'ga !== void 0 ? ga(\'send\', \'event\', \'' + accommodation.title + '\', \'card\', \'uitgaand bezoek booking button\') : null'">
                    {{ t('Boek nu') }}
                </a>
            </div>

            <div class="h-accommodation-buttons" v-if="!!category || (!category && !bookNowButtons && !onlySelected)">
                <a class="h-button h-button-reservation" :href="pandora ? pandora : ((item.address && item.address.website) || item.website)" :target="pandora ? null : '_blank'" @click="viewItem(item, $event, 'addToCart')" :onclick="pandora ? null : (getTrackEvent('website') || 'ga !== void 0 ? ga(\'send\', \'event\', \'' + item.title + '\', \'card\', \'uitgaand bezoek website button\') : null')" v-if="button">
                    <span class="h-button-label">{{ bookLabel ? bookLabel : t('Reserveer direct') }}</span>
                </a>
                <a class="h-button h-button-telephone" :href="'tel:' + item.address.telephone" v-if="item.address && item.address.telephone"></a>
                <a class="h-button h-button-view" v-if="item.url && item.url.length" :href="item.url" @click="viewItem(item, $event, 'viewing')">
                    {{ t('Nu bekijken') }}
                </a>
            </div>

            <div class="h-accommodation-buttons" v-if="onlySelected">
                <a class="h-button" v-if="!(selected == item.id)" @click="$emit('selected', {accommodation: item.id})">
                    <span class="h-button-label">{{ t('Selecteer') }}</span>
                </a>
                <a class="h-button" v-else @click="$emit('selected', {accommodation: null})">
                    <span class="h-button-label">{{ t('Wijzig uw keuze') }}</span>
                </a>
            </div>
            <div class="h-accommodation-buttons h-accommodation-button-info" v-if="moreInfo">
                <a href="#" class="h-button h-button-more" @click="$event.preventDefault(); show.more = true; show.moreDelay = true; show.packages = false; show.price = false">
                  {{ t('Meer info') }}
                </a>
            </div>

            <div v-if="item.price && showPrice && (item.available === false || alternativePrices)" class="h-accommodation-price h-accommodation-price-matrix" :class="{ open: show.rhea === item.id }" @click="checkClickOverlay($event)">
                <a :href="item.url" target="_blank" @click="$event.preventDefault(); show.rhea = item.id; scrollRhea()">{{ t('Bekijk beschikbaarheid') }}</a>

                <div class="h-accommodation-rhea" v-show="show.rhea === item.id" ref="rhea">
                    <rhea v-if="show.rhea === item.id" :accommodation-id="accommodation ? accommodation.id : null" :category-id="category ? category.id : null" :small="true" :show-price="false" :period-id="period ? period.id : null" :first="format(item.price && item.price.arrival ? item.price.arrival : arrival)" />
                </div>
            </div>

            <div class="h-accommodation-packages" :class="{ open: show.packages }" @click="checkClickOverlay($event)" v-if="item.packages && item.packages.length > 0" ref="packages">
                <button class="h-button" @click="show.packages = !show.packages; scrollPackages()">
                    {{ t('Bekijk arrangementen') }} <span class="h-badge">{{ item.packages.length }}</span>
                </button>

                <packages v-show="show.packages" :showing="show.packages" :accommodation="subItem" :packages="item.packages" @close="() => this.show.packages = false" />
            </div>
            <div class="h-accommodation-accommodations" :class="{ open: show.accommodations }" @click="checkClickOverlay($event)" v-if="category && !showPrice && showAccommodations" ref="telesto">
                <button class="h-button" @click="show.accommodations = !show.accommodations; scrollTelesto()">
                    {{ t('Bekijk mogelijkheden') }} <span class="h-badge">{{ item.accommodations.length }}</span>
                </button>

                <div class="harbor-widget harbor-widget-telesto" :class="{ 'h-has-more': showAccommodations && item.accommodations && item.accommodations.length >= 6 }" v-show="show.accommodations">
                    <button type="button" class="h-close" @click="show.accommodations = false"></button>

                    <div class="h-accommodations">
                        <div class="h-columns">
                            <div class="h-column" v-for="accommodation in item.accommodations" :key="accommodation.id">
                                <accommodation-alt :accommodation="accommodation" :show-price="false" :show-accommodations="false" :website="item.address && item.address.website" :more-info="moreInfo" :bookNowButtons="bookNowButtons" />
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
  import format from 'date-fns/format';
  import diffInDays from 'date-fns/difference_in_days';

  import metaQuery from '../queries/meta.gql';
  import Packages from './Packages';
  import Price from './Price';
  import { url } from '../plugins/pandora';
  import bus from '../plugins/bus';

  export default {
    props: {
        categoryId: {
            type: Number,
            default: null,
        },
        position: {
            type: Number,
            default: null
        },
      category: {
        type: Object,
        default() {
          return null;
        },
      },

      accommodation: {
        type: Object,
        default() {
          return null;
        },
      },

      period: {
        type: Object,
        default() {
          return null;
        },
      },

      arrival: {
        type: String,
        default: null,
      },

      departure: {
        type: String,
        default: null,
      },

      people: {
        type: Object,
        default: null,
      },

      url: {
        type: String,
        default: null,
      },

      showPrice: {
        type: Boolean,
        default: true,
      },

      showPersonPrice: {
        type: Boolean,
        default: false,
      },

      showTags: {
        type: Boolean,
        default: false,
      },

      showDates: {
        type: Boolean,
        default: false,
      },

      button: {
        type: Boolean,
        default: true,
      },

      moreInfo: {
        type: Boolean,
        default: false,
      },

      showAccommodations: {
        type: Boolean,
        default: true,
      },

      bookNowButtons: {
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

      infoHeight: {
        type: Number,
        default: null,
      },

      moreInfoLinkTarget: {
        type: String,
        default: '_blank'
      },

      onlySelected: {
        type: Boolean,
        default: false
      },

      selected: {
        type: Number,
        default: null
      },

      bookNowReserveButton: {
        type: Boolean,
        default: true
      },

      bookNowMoreInfoButton: {
        type: Boolean,
        default: true
      },

      depositBelowPrice: {
        type: Boolean,
        default: true
      },

      priceNewStructure: {
        type: Boolean,
        default: false
      },

      bookLabel: {
        type: String,
        default: null
      },

      expandableInformation: {
        type: Boolean,
        default: false
      },

      subTitleOnTop: {
        type: Boolean,
        default: false
      },

      alternativePrices: {
        type: Boolean,
        default: false
      },
      
      showAccommodationOptions: {
        type: Boolean,
        default: false
      },
    },

    components: {
      accommodationAlt: require('./AccommodationAlt'),
      packages: Packages,
      price: Price,
    },

    apollo: {
      settingsPeople: {
        query: metaQuery,

        update: (data) => data.people,
      },
    },

    data() {
      return {
        isTouch: typeof window !== void 0 && 'ontouchstart' in window,

        carousel: {
          resistanceCoef: 20,
          current: 0,
          offset: 0,

          drag: {
            dragging: false,
            offset: 0,
            x: 0,
            y: 0,
          },
        },

        settingsPeople: [],

        show: {
          more: false,
          moreDelay: false,
          packages: false,
          price: false,
          rhea: null,
          accommodations: false,
          moreInfo: false
        },

        truncated: {
          information: false,
          accommodations: []
        },

        initialHeight: null,

        expandInfo: false,

        infoExpandedId: null,

        selection: {
          accommodation: this.category ? this.category.accommodations[0].id : null
        }
      };
    },

    computed: {
      item() {
        if (this.category) {
          return {
            tags: [],
            packages: (this.category.accommodations || []).map((item) => (
              item.packages
            )).flat(1).filter((item) => !!item),
            ...this.category,
          };
        }

        return {
          tags: [],
          ...this.accommodation
        };
      },

      subItem() {
        if (this.item && this.item.price && this.category) {
          return this.category.accommodations.find((item) => (
            item.id == this.item.price.accommodation
          ))
        }

        // if (this.item && this.item.price && this.item.price.accommodation) {
        //   return this.category.accommodations.find((item) => (
        //     item.id == this.item.price.accommodation
        //   ))
        // }

        return null;
      },

      style() {
        if (this.show.packages) {
          return {
            zIndex: 10,
          };
        }
      },

      nights() {
        const diff = diffInDays(this.departure, this.arrival);

        return diff > 0 ? diff : 0;
      },

      slideWidth() {
        const dragging = this.carousel.drag.dragging;
        const current = this.carousel.current;

        if (!this.$refs.images) {
          return 0;
        }

        const image = this.$refs.images.querySelector('.h-accommodation-image');

        return image ? image.offsetWidth : 0;
      },

      slideOffset() {
        const current = (this.carousel.current * this.slideWidth);

        return -(this.carousel.drag.dragging ? (current + this.carousel.drag.offset) : (this.carousel.current * this.slideWidth));
      },

      pandora() {
        if (!this.url || this.url === '?') {
          return null;
        }

        let item = this.subItem ? this.subItem : this.item

        if (this.selection.accommodation) {
          item = this.category.accommodations.find(a => a.id === this.selection.accommodation)
        }

        return url(this.url, (this.item.price && this.item.price.arrival) || this.arrival, (this.item.price && this.item.price.departure) || this.departure, this.people, this.settingsPeople, item, null, this.period);
      },
    },

    mounted() {
      this.$refs.basic.addEventListener('transitionend', () => {
        this.show.moreDelay = this.show.more;
      });

      this.$refs.images.addEventListener(
        this.isTouch ? 'touchstart' : 'mousedown', this.onStart
      );

      this.$emit('info-height', this.$refs.accommodationInformation ? this.$refs.accommodationInformation.clientHeight : null);

      if (this.expandableInformation) {
        this.checkForTruncated();

        bus.$on('accommodation-info-expanded', (id) => {
          this.infoExpandedId = id;
        });

        bus.$on('reset-expanded-information', () => {
          this.infoExpandedId = null;
        });
      }

      if (this.showAccommodationOptions) {
        this.checkForTruncatedAccommodationLabel();
      }
    },

    watch: {
      'show.moreInfo'(value) {
        if (value) {
          bus.$emit('accommodation-info-expanded', this.item.id);
        } else {
          this.expandInfo = false;
        }
      },

      expandInfo(value) {
        if (this.$refs.accommodationInformation) {
          const span = this.$refs.accommodationInformation.querySelector('span');
          if (span) {

            span.style.maxHeight = value ? span.scrollHeight + 'px' : '';

            if (!value) {
              span.addEventListener('transitionend', () => {
                this.$refs.accommodation.style.height = this.initialHeight + 'px';

                bus.$emit('accommodation-info-collapsed', this.item.id);
              }, {once: true});
            }
          }
        }

        if (value) {
          this.$refs.accommodation.style.height = '100%';
        }
      },

      infoExpandedId(id, oldId) {
        const isOther = id !== this.item.id;

        if (isOther) {
          this.expandInfo = this.show.moreInfo = false;
        } else if (this.show.moreInfo) {
          if (oldId) {
            bus.$once('accommodation-info-collapsed', (collapsedId) => {
              this.expandInfo = true;
            });
          } else if (this.infoExpandedId && this.infoExpandedId === this.item.id) {
            this.expandInfo = true;
          }
        }
      }
    },

    methods: {
      format(date, formatDate = 'YYYY-MM-DD') {
        return format(date, formatDate);
      },

      checkClickOverlay(e) {
        if (e.target.classList.contains('open') && e.target.classList.contains('h-accommodation-packages')) {
          this.show.packages = false;
        }

        if (e.target.classList.contains('open') && e.target.classList.contains('h-accommodation-accommodations')) {
          this.show.accommodations = false;
        }

        if (e.target.classList.contains('open') && e.target.classList.contains('h-accommodation-price')) {
          this.show.rhea = null;
        }
      },

      viewItem(item, e, action) {
        const selected = {
          ...this.$storage.retrieve('selected'),
          accommodation: this.showAccommodationOptions ? this.selection.accommodation : item.id,
          package: null,
        };

        if (this.item.price) {
          if (this.item.price.arrival) {
            selected.arrival = this.item.price.arrival;
          }

          if (this.item.price.departure) {
            selected.departure = this.item.price.departure;
          }

          if (this.item.price.period) {
            selected.period = this.item.price.period;
          }
        }

        if (this.category && item.accommodations && item.accommodations.length > 0) {
          if (this.showAccommodationOptions) {
            selected.accommodation = this.selection.accommodation;
          } else if (item.price && item.price.accommodation) {
            selected.accommodation = item.price.accommodation;
          } else {
            selected.accommodation = item.accommodations[0].id;
          }
        }

        this.$storage.saveSession('book', true);

        this.$storage.save('selected', selected);
        this.$emit('go-out', item, e);

        if(item){
            var cat = this.categoryId ? item.categories.find((value) => {return value.id === this.categoryId}) : null
            cat = cat ? cat.title : (this.item.title ? this.item.title : null);
            if(action ==='viewing'){
                this.$ga.ecommerce.addProduct({
                    'name': item.title,
                    'id': item.provider.key,
                    'price': item.price.total,
                    'category': cat ? cat : null,
                    'list': 'Overview',
                    'position': this.position
                });
                this.$ga.ecommerce.setAction('click')
                this.$ga.event('UX', 'click', 'Results');
            }else if(action ==='addToCart'){
                this.$ga.ecommerce.addProduct({
                    'name': item.title,
                    'id': item.provider.key,
                    'price': item.price.total,
                    'category': cat ? cat : null,
                    'quantity': 1
                });
                this.$ga.ecommerce.setAction('add')
                this.$ga.event('UX', 'click', 'add to cart');
            }
        }
      },

      scrollPackages() {
        const height = Math.max(document.documentElement.clientHeight, window.innerHeight || 0);

        this.$nextTick(() => {
          const packages = this.$refs.packages.querySelector('.harbor-widget-packages');
          let top = this.$refs.packages.getBoundingClientRect().top - (height / 2) + (packages.offsetHeight / 2);

          if (height < packages.offsetHeight) {
            top = this.$refs.packages.getBoundingClientRect().top
          }

          const rect = packages.getBoundingClientRect();
          const inViewport = (
              rect.top >= 0 &&
              rect.left >= 0 &&
              rect.bottom <= (window.innerHeight || document.documentElement.clientHeight) &&
              rect.right <= (window.innerWidth || document.documentElement.clientWidth)
          );

          if (inViewport) {
            top = 0;
          }

          window.scrollBy({
            top,
            behavior: 'smooth',
          });
        });
      },

      scrollRhea() {
        const height = Math.max(document.documentElement.clientHeight, window.innerHeight || 0);

        this.$nextTick(() => {
          const rhea = this.$refs.rhea.querySelector('.harbor-widget-rhea');
          let top = this.$refs.rhea.getBoundingClientRect().top - (height / 2) + (rhea.offsetHeight / 2);

          if (height < rhea.offsetHeight) {
            top = this.$refs.rhea.getBoundingClientRect().top
          }

          const rect = rhea.getBoundingClientRect();
          const inViewport = (
              rect.top >= 0 &&
              rect.left >= 0 &&
              rect.bottom <= (window.innerHeight || document.documentElement.clientHeight) &&
              rect.right <= (window.innerWidth || document.documentElement.clientWidth)
          );

          if (inViewport) {
            top = 0;
          }

          window.scrollBy({
            top,
            behavior: 'smooth',
          });
        });
      },

      scrollTelesto() {
        const height = Math.max(document.documentElement.clientHeight, window.innerHeight || 0);

        this.$nextTick(() => {
          const telesto = this.$refs.telesto.querySelector('.harbor-widget-telesto');
          let top = this.$refs.telesto.getBoundingClientRect().top - (height / 2) + (telesto.offsetHeight / 2);

          if (height < telesto.offsetHeight) {
            top = this.$refs.telesto.getBoundingClientRect().top
          }

          const rect = telesto.getBoundingClientRect();
          const inViewport = (
              rect.top >= 0 &&
              rect.left >= 0 &&
              rect.bottom <= (window.innerHeight || document.documentElement.clientHeight) &&
              rect.right <= (window.innerWidth || document.documentElement.clientWidth)
          );

          if (inViewport) {
            top = 0;
          }

          window.scrollBy({
            top,
            behavior: 'smooth',
          });
        });
      },

      getTrackEvent(label, item = null) {
        if (!this.item.events) {
          return null;
        }

        const event = this.item.events.find((event) => (
          event.label == label
        ))
        if(item){
            var cat = this.categoryId ? item.categories.find((value) => {return value.id === this.categoryId}) : null
            if(label === 'booking' && item && item.price.total && item.title && item.provider.key){
                this.$ga.ecommerce.addProduct({
                    'name': item.title,
                    'id': item.provider.key,
                    'category': cat ? cat.title : null,
                    'price': item.price.total,
                    'quantity': 1
                });
                this.$ga.ecommerce.setAction('add')
                this.$ga.event('UX', 'click', 'add to cart');
            }else if(label === 'website' && item && item.price.total && item.title && item.provider.key){
                this.$ga.ecommerce.addProduct({
                    'name': item.title,
                    'id': item.provider.key,
                    'category': cat ? cat.title : null,
                    'price': item.price.total,
                    'quantity': 1
                });
                this.$ga.ecommerce.setAction('click')
                this.$ga.event('UX', 'click', 'Results');
            }
        }

        return event ? event.value : null;
      },

      //

      onStart(e) {
        document.addEventListener(
          this.isTouch ? 'touchend' : 'mouseup', this.onEnd, true
        );

        if (!this.item.images || this.item.images.length === 1) {
          return;
        }

        document.addEventListener(
          this.isTouch ? 'touchmove' : 'mousemove', this.onDrag, true
        );

        this.carousel.drag = {
          dragging: true,
          offset: 0,
          x: this.isTouch ? e.touches[0].clientX : e.clientX,
          y: this.isTouch ? e.touches[0].clientY : e.clientY,
        };
      },

      onDrag(e) {
        const offset = {
          x: this.carousel.drag.x - (this.isTouch ? e.touches[0].clientX : e.clientX),
          y: this.carousel.drag.y - (this.isTouch ? e.touches[0].clientY : e.clientY),
        };

        if (this.isTouch && Math.abs(offset.x) < Math.abs(offset.y)) {
          return;
        }

        e.stopImmediatePropagation();

        this.carousel.drag.offset = offset.x;

        if ((-this.slideOffset) < 0) {
          this.carousel.drag.offset = -Math.sqrt(-this.carousel.resistanceCoef * this.carousel.drag.offset);
        } else if ((-this.slideOffset) > ((this.item.images.length - 1) * this.slideWidth)) {
          this.carousel.drag.offset = Math.sqrt(this.carousel.resistanceCoef * this.carousel.drag.offset);
        }
      },

      onEnd(e) {
        document.removeEventListener(
          this.isTouch ? 'touchend' : 'mouseup', this.onEnd, true
        );

        document.removeEventListener(
          this.isTouch ? 'touchmove' : 'mousemove', this.onDrag, true
        );

        if (this.carousel.drag.offset === 0 && !this.isTouch) {
          this.viewItem(this.item, null, 'viewing');

          window.location.href = this.item.url;
        }

        let slide = Math.round(((-this.slideOffset) / this.slideWidth));

        if (slide < 0) {
          slide = 0;
        }

        if (slide > this.item.images.length - 1) {
          slide = this.item.images.length - 1;
        }

        this.carousel.drag.dragging = false;
        this.carousel.current = slide;
      },

      filterFeatures(features) {
        return features.filter((feature) => (typeof feature  === 'object' && feature.title));
      },

      checkForTruncated() {
        this.$nextTick(() => {
          if (this.$refs.accommodationInformation) {
            const span = this.$refs.accommodationInformation.querySelector('span');
            if (span) {
              this.checkTooltip(span);
            }
          }

          if (this.$refs.accommodation) {
            this.initialHeight = this.$refs.accommodation.clientHeight;

            this.$refs.accommodation.style.height = this.initialHeight + 'px';
          }
        });
      },

      checkTooltip(e) {
        if (e.clientHeight >= e.scrollHeight) {
          return;
        } else if ((e.clientHeight + 1) >= e.scrollHeight) {
          return;
        }

        if (!this.truncated.information) {
          this.truncated.information = true;
        }
      },

      toggleShowInfo() {
        this.show.moreInfo = !this.show.moreInfo;

        if (!this.show.moreInfo) {
          bus.$emit('reset-expanded-information');
        }
      },

      checkTooltipAccommodationLabel(e, i) {
        if (e.offsetWidth >= e.scrollWidth) {
          return;
        }

        if (this.truncated.accommodations.indexOf(i) === -1) {
          this.truncated.accommodations.push(i);
        }
      },

      checkForTruncatedAccommodationLabel() {
        this.$nextTick(() => {
          console.log(this.$refs.accommodationOptions);
          if (this.$refs.accommodationOptions) {
            this.$refs.accommodationOptions.forEach((e, i) => {
              this.checkTooltipAccommodationLabel(e.querySelector('.h-title-truncated'), 'a' + i)
            });
          }
        });
      },
    },
  };
</script>
