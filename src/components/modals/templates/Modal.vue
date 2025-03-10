<template>
  <Teleport to="body">
    <div
      class="fixed flex top-0 bottom-0 left-0 right-0 justify-center background/70 z-[999999999] modal-send-receive-parent"
      @keydown.esc="onModalClose"
      ref="dialog"
    >
      <button
        v-if="collpase"
        class="btn-close-modal"
        @click="onModalClose"
      >
        <svg
          width="28"
          height="28"
          viewBox="0 0 28 28"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            fill-rule="evenodd"
            clip-rule="evenodd"
            d="M5.02514 15.0333C5.02517 14.8124 5.11238 14.6006 5.26758 14.4444C5.42278 14.2882 5.63327 14.2004 5.85276 14.2004H12.8764C13.0959 14.2004 13.3064 14.2882 13.4616 14.4444C13.6168 14.6006 13.704 14.8124 13.704 15.0333L13.704 22.1021C13.7059 22.2127 13.686 22.3226 13.6452 22.4253C13.6045 22.5281 13.5438 22.6216 13.4668 22.7005C13.3897 22.7794 13.2979 22.8421 13.1965 22.8848C13.0951 22.9276 12.9863 22.9496 12.8764 22.9496C12.7665 22.9496 12.6577 22.9276 12.5563 22.8848C12.455 22.8421 12.3631 22.7794 12.286 22.7005C12.209 22.6216 12.1484 22.5281 12.1076 22.4253C12.0669 22.3226 12.0469 22.2127 12.0488 22.1021L12.0488 17.0444L5.26746 23.8693C5.11222 24.0255 4.90168 24.1133 4.68215 24.1133C4.46262 24.1133 4.25208 24.0255 4.09685 23.8693C3.94161 23.7131 3.85441 23.5012 3.85441 23.2802C3.85441 23.0593 3.94161 22.8474 4.09685 22.6912L10.8782 15.8663L5.85276 15.8663C5.63327 15.8662 5.42278 15.7785 5.26758 15.6223C5.11238 15.4661 5.02517 15.2542 5.02514 15.0333Z"
            fill="currentColor"
          />
          <path
            fill-rule="evenodd"
            clip-rule="evenodd"
            d="M22.9749 12.9667C22.9748 13.1876 22.8876 13.3994 22.7324 13.5556C22.5772 13.7118 22.3667 13.7996 22.1472 13.7996L15.1236 13.7996C14.9041 13.7996 14.6936 13.7118 14.5384 13.5556C14.3832 13.3994 14.296 13.1876 14.296 12.9667L14.296 5.89789C14.2941 5.78729 14.314 5.67742 14.3548 5.57467C14.3955 5.47193 14.4562 5.37839 14.5332 5.29949C14.6103 5.2206 14.7021 5.15793 14.8035 5.11516C14.9049 5.07238 15.0137 5.05035 15.1236 5.05035C15.2335 5.05035 15.3423 5.07238 15.4437 5.11516C15.545 5.15793 15.6369 5.2206 15.714 5.29949C15.791 5.37839 15.8516 5.47193 15.8924 5.57468C15.9331 5.67742 15.9531 5.78729 15.9512 5.89789L15.9512 10.9556L22.7325 4.1307C22.8878 3.97447 23.0983 3.8867 23.3178 3.8867C23.5374 3.8867 23.7479 3.97447 23.9032 4.1307C24.0584 4.28693 24.1456 4.49882 24.1456 4.71976C24.1456 4.94071 24.0584 5.1526 23.9032 5.30883L17.1218 12.1337L22.1472 12.1337C22.3667 12.1338 22.5772 12.2215 22.7324 12.3777C22.8876 12.5339 22.9748 12.7458 22.9749 12.9667Z"
            fill="currentColor"
          />
        </svg>

      </button>
      <button
        v-else
        class="btn-close-modal"
        @click="onModalClose"
      >
        <XMarkIcon class="inline-block w-8 h-8 z-[5]" />
      </button>
      <slot></slot>
    </div>
  </Teleport>
</template>

<script lang="ts" setup>
import { onMounted, onUnmounted, provide, ref } from "vue";
import { XMarkIcon } from "@heroicons/vue/24/solid";
import router from "@/router";

const collpase = ref(false);
const dialog = ref<HTMLDivElement>();
const props = defineProps({
  disableClose: {
    type: Boolean
  },
  route: {
    type: String,
    default: "dialog",
  },
});

const emit = defineEmits(["close-modal"]);

const setCollapseButton = (bool: boolean) => {
  collpase.value = bool;
};

const onModalClose = () => {
  if(!props.disableClose){
    parseRoute();
    close();
  }else{
    emit("close-modal");
  }
};

const parseRoute = () => {
  const path = router.currentRoute.value.path;
  router.push({
    path,
  });
};

const escapeClicked = (event: KeyboardEvent) => {
  if (event.key == "Escape") {
    onModalClose();
  }
};

const backButtonClicked = (event: Event) => {
  close();
};

onMounted(() => {
  document.body.style.overflowY = "hidden";

  const element = dialog.value;
  if (element) {
    element.style.animation = 'fadeInAnimation 200ms';
  }

  const path = router.currentRoute.value.path;
  router.push({
    path,
    hash: `#${props.route.toLowerCase()}`,
  });
  document.addEventListener("keyup", escapeClicked);
  window.addEventListener("popstate", backButtonClicked);

});

onUnmounted(() => {
  document.removeEventListener("keyup", escapeClicked);
  window.removeEventListener("popstate", backButtonClicked);
});

const close = () => {
  const element = dialog.value;
  if (element) {
    element.style.animation = 'fadeOutAnimation 200ms';
  }
  setTimeout(() => {
    emit("close-modal");
    document.body.style.overflowY = "auto";
  }, 100);
}

provide("onModalClose", onModalClose);
provide("parseRoute", parseRoute);
provide("setCollapseButton", setCollapseButton);

defineExpose({ onModalClose })
</script>
