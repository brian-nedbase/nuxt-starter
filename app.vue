<script setup lang="ts">
import type { Schemas } from "#shopware";
import { getPrefix } from "./i18n/src/helpers/prefix";
const config = useRuntimeConfig();
/**
 * Init breadcrumbs context
 */
useBreadcrumbs();

useHead({
  title: "Shopware Demo store",
  meta: [{ name: "description", content: "Shopware Demo store" }],
  link: [
    { rel: 'manifest', href: '/manifest.webmanifest' }
  ],
  htmlAttrs: {
    lang: "en",
  },
});

const { apiClient } = useShopwareContext();
const sessionContextData = ref<Schemas["SalesChannelContext"]>();
const contextResponse = await apiClient.invoke("readContext get /context");
sessionContextData.value = contextResponse.data;

if (config.public.broadcasting) {
  useBroadcastChannelSync();
}

// If you enable runtimeConfig.shopware.useUserContextInSSR, then you can use this code to share session between server and client.
// const { data: sessionContextData } = await useAsyncData(
//   "sessionContext",
//   async () => {
//     return await getSessionContext(apiInstance);
//   }
// );

usePrice({
  currencyCode: sessionContextData.value?.currency?.isoCode || "",
});

useSessionContext(sessionContextData.value);

const { getWishlistProducts } = useWishlist();
const { refreshCart } = useCart();

useNotifications();
useAddress();

const { locale, availableLocales, defaultLocale, localeProperties, messages } =
    useI18n();

const router = useRouter();
const {
  getAvailableLanguages,
  getLanguageCodeFromId,
  getLanguageIdFromCode,
  changeLanguage,
  languages: storeLanguages,
} = useInternationalization();
const { languageIdChain, refreshSessionContext } = useSessionContext();

const { data: languages } = await useAsyncData("languages", async () => {
  return await getAvailableLanguages();
});
let languageToChangeId: string | null = null;

if (languages.value?.elements.length && router.currentRoute.value.name) {
  storeLanguages.value = languages.value?.elements;
  // Prefix from url
  const prefix = getPrefix(
      availableLocales,
      router.currentRoute.value.name as string,
      defaultLocale,
  );

  provide(
      "cmsTranslations",
      messages.value[(prefix as keyof typeof messages.value) || defaultLocale] ??
      {},
  );

  // Language set on the backend side
  if (localeProperties.value.localeId) {
    if (languageIdChain.value !== localeProperties.value.localeId) {
      languageToChangeId = localeProperties.value.localeId;
    }
  } else {
    const sessionLanguage = getLanguageCodeFromId(languageIdChain.value);

    // If languages are not the same, set one from prefix
    if (sessionLanguage !== prefix) {
      languageToChangeId = getLanguageIdFromCode(
          prefix ? prefix : defaultLocale,
      );
    }
  }

  if (languageToChangeId) {
    apiClient.defaultHeaders.apply({
      "sw-language-id": languageToChangeId,
    });
    await changeLanguage(languageToChangeId);
    await refreshSessionContext();
  }

  locale.value = (
      prefix ? prefix : defaultLocale
  ) as keyof typeof messages.value;
  // Set prefix from CMS components
  provide("urlPrefix", prefix);
}

if (import.meta.client) {
  // getting the wishlist products should not block SSR
  if (!(router.currentRoute.value.name as string).includes("wishlist")) {
    getWishlistProducts(); // initial page loading
  }
}

onMounted(() => {
  refreshCart();
});
</script>

<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>

<style>
h2 {
  @apply text-4xl py-4;
}

select {
  background-image: url("data:image/svg+xml;base64,PHN2ZyB4bWxucz0naHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmcnIGZpbGw9J25vbmUnIHZpZXdCb3g9JzAgMCAyMCAyMCc+PHBhdGggc3Ryb2tlPScjNmI3MjgwJyBzdHJva2UtbGluZWNhcD0ncm91bmQnIHN0cm9rZS1saW5lam9pbj0ncm91bmQnIHN0cm9rZS13aWR0aD0nMS41JyBkPSdNNiA4bDQgNCA0LTQnLz48L3N2Zz4=");
  background-position: right 0.5rem center;
  background-repeat: no-repeat;
  background-size: 1.5em 1.5em;
  appearance: none;
}
</style>
