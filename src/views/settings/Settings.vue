<template lang="pug">
div
  h3 {{ $t('settings') }}

  b-alert(variant="warning", show) #[b Note:] {{ $t('settingsHelp') }}

  hr

  DaystartSettings

  hr

  TimelineDurationSettings

  hr

  LandingPageSettings

  hr

  //- div(v-if="!$isAndroid")
  //-   ReleaseNotificationSettings
  //-   hr

  CategorizationSettings

</template>

<script>
import DaystartSettings from '~/views/settings/DaystartSettings.vue';
import TimelineDurationSettings from '~/views/settings/TimelineDurationSettings.vue';
// import ReleaseNotificationSettings from '~/views/settings/ReleaseNotificationSettings.vue';
import CategorizationSettings from '~/views/settings/CategorizationSettings.vue';
import LandingPageSettings from '~/views/settings/LandingPageSettings.vue';

export default {
  name: 'Settings',
  components: {
    DaystartSettings,
    TimelineDurationSettings,
    // ReleaseNotificationSettings,
    CategorizationSettings,
    LandingPageSettings,
  },
  data() {
    return {
      isAndroid: process.env.VUE_APP_ON_ANDROID,
    };
  },
};
</script>
