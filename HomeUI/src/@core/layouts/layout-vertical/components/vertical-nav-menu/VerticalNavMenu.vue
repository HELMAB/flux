<template>
  <div
    class="main-menu menu-fixed menu-accordion menu-shadow"
    :class="[
      { 'expanded': !isVerticalMenuCollapsed || (isVerticalMenuCollapsed && isMouseHovered) },
      skin === 'semi-dark' ? 'menu-dark' : 'menu-light'
    ]"
    @mouseenter="updateMouseHovered(true)"
    @mouseleave="updateMouseHovered(false)"
  >
    <!-- main menu header-->
    <div class="navbar-header expanded">
      <slot
        name="header"
        :toggleVerticalMenuActive="toggleVerticalMenuActive"
        :toggleCollapsed="toggleCollapsed"
        :collapseTogglerIcon="collapseTogglerIcon"
      >
        <ul class="nav navbar-nav flex-row">
          <!-- Logo & Text -->
          <li class="nav-item mr-auto">
            <b-link
              class="navbar-brand"
              to="/"
            >
              <span class="brand-logo">
                <b-img
                  :src="skin === 'dark' ? appLogoImageDark : appLogoImage"
                  alt="logo"
                  :class="(isVerticalMenuCollapsed ? 'collapsed-logo' : '')"
                />
              </span>
              <!-- <h2 class="brand-text">
                {{ appName }}
              </h2> -->
            </b-link>
          </li>

          <!-- Toggler Button -->
          <li class="nav-item nav-toggle">
            <b-link class="nav-link modern-nav-toggle">
              <feather-icon
                icon="XIcon"
                size="20"
                class="d-block d-xl-none"
                @click="toggleVerticalMenuActive"
              />
              <feather-icon
                :icon="collapseTogglerIconFeather"
                size="20"
                class="d-none d-xl-block collapse-toggle-icon"
                @click="toggleCollapsed"
              />
            </b-link>
          </li>
        </ul>
      </slot>
    </div>
    <!-- / main menu header-->

    <!-- Shadow -->
    <div
      :class="{'d-block': shallShadowBottom}"
      class="shadow-bottom"
    />

    <!-- main menu content-->
    <vue-perfect-scrollbar
      :settings="perfectScrollbarSettings"
      class="main-menu-content scroll-area"
      tagname="ul"
      @ps-scroll-y="evt => { shallShadowBottom = evt.srcElement.scrollTop > 0 }"
    >
      <vertical-nav-menu-items
        :key="isNavMenuCollapsed"
        :items="isNavMenuCollapsed ? navMenuItemsCollapsed : navMenuItems"
        class="navigation navigation-main"
      />
    </vue-perfect-scrollbar>
    <!-- /main menu content-->
  </div>
</template>

<script>
import VuePerfectScrollbar from 'vue-perfect-scrollbar';
import { BLink, BImg } from 'bootstrap-vue';
import {
  provide,
  computed,
  ref,
  onBeforeMount,
} from '@vue/composition-api';
import useAppConfig from '@core/app-config/useAppConfig';
import { $themeConfig } from '@themeConfig';
import navMenuItems from '@/navigation/vertical';
import navMenuItemsCollapsed from '@/navigation/vertical/index_collapsed';
import VerticalNavMenuItems from './components/vertical-nav-menu-items/VerticalNavMenuItems.vue';
import useVerticalNavMenu from './useVerticalNavMenu';

const qs = require('qs');
const axios = require('axios');

export default {
  components: {
    VuePerfectScrollbar,
    VerticalNavMenuItems,
    BLink,
    BImg,
  },
  props: {
    isVerticalMenuActive: {
      type: Boolean,
      required: true,
    },
    toggleVerticalMenuActive: {
      type: Function,
      required: true,
    },
  },
  setup(props) {
    const {
      isMouseHovered,
      isVerticalMenuCollapsed,
      collapseTogglerIcon,
      toggleCollapsed,
      updateMouseHovered,
    } = useVerticalNavMenu(props);

    const zelid = ref(null);

    onBeforeMount(() => {
      const zelidauth = localStorage.getItem('zelidauth');
      const auth = qs.parse(zelidauth);
      zelid.value = auth.zelid;
    });

    const {
      isNavMenuCollapsed,
      xdaoOpenProposals,
      skin,
    } = useAppConfig();

    const getVoteInformation = async (proposal) => {
      const response = await axios.get(`https://stats.runonflux.io/proposals/voteInformation?hash=${proposal.hash}&zelid=${zelid.value}`);
      return response.data;
    };

    const checkXDAOProposals = async () => {
      let openNotVoted = 0;
      axios.get('https://stats.runonflux.io/proposals/listProposals').then((response) => {
        if (response.data.status === 'success') {
          const openProposals = response.data.data.filter((proposal) => proposal.status === 'Open');
          openProposals.forEach(async (proposal) => {
            const voteInformation = await getVoteInformation(proposal);
            if (voteInformation.status === 'success' && (voteInformation.data == null || voteInformation.data.length === 0)) {
              openNotVoted += 1;
              xdaoOpenProposals.value = openNotVoted;
            }
          });
        }
      });
    };

    setInterval(() => {
      checkXDAOProposals();
    }, 1000 * 60 * 10); // Refresh every 10 minutes
    checkXDAOProposals();

    // Shadow bottom is UI specific and can be removed by user => It's not in `useVerticalNavMenu`
    const shallShadowBottom = ref(false);

    provide('isMouseHovered', isMouseHovered);

    const perfectScrollbarSettings = {
      maxScrollbarLength: 60,
      wheelPropagation: false,
    };

    const collapseTogglerIconFeather = computed(() => (collapseTogglerIcon.value === 'unpinned' ? 'CircleIcon' : 'DiscIcon'));

    // App Name
    const { appName, appLogoImageDark, appLogoImage } = $themeConfig.app;

    return {
      navMenuItems,
      navMenuItemsCollapsed,
      perfectScrollbarSettings,
      isVerticalMenuCollapsed,
      collapseTogglerIcon,
      toggleCollapsed,
      isMouseHovered,
      updateMouseHovered,
      collapseTogglerIconFeather,

      // Shadow Bottom
      shallShadowBottom,

      // Skin
      skin,
      isNavMenuCollapsed,

      // App Name
      appName,
      appLogoImage,
      appLogoImageDark,
    };
  },
};
</script>

<style lang="scss">
@import "~@core/scss/base/core/menu/menu-types/vertical-menu.scss";
</style>
