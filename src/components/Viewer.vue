<template>
  <div
    class="memories_viewer outer"
    v-if="show"
    :class="{ fullyOpened }"
    :style="{ width: outerWidth }"
  >
    <ImageEditor
      v-if="editorOpen"
      :mime="currentPhoto.mimetype"
      :src="currentDownloadLink"
      :fileid="currentPhoto.fileid"
      @close="editorOpen = false"
    />

    <div
      class="inner"
      ref="inner"
      v-show="!editorOpen"
      @pointermove.passive="setUiVisible"
      @pointerdown.passive="setUiVisible"
    >
      <div class="top-bar" v-if="photoswipe" :class="{ showControls }">
        <NcActions
          :inline="numInlineActions"
          container=".memories_viewer .pswp"
        >
          <NcActionButton
            v-if="canShare"
            :aria-label="t('memories', 'Share')"
            @click="shareCurrent"
            :close-after-click="true"
          >
            {{ t("memories", "Share") }}
            <template #icon> <ShareIcon :size="24" /> </template>
          </NcActionButton>
          <NcActionButton
            v-if="!routeIsPublic"
            :aria-label="t('memories', 'Delete')"
            @click="deleteCurrent"
            :close-after-click="true"
          >
            {{ t("memories", "Delete") }}
            <template #icon> <DeleteIcon :size="24" /> </template>
          </NcActionButton>
          <NcActionButton
            v-if="!routeIsPublic"
            :aria-label="t('memories', 'Favorite')"
            @click="favoriteCurrent"
            :close-after-click="true"
          >
            {{ t("memories", "Favorite") }}
            <template #icon>
              <StarIcon v-if="isFavorite()" :size="24" />
              <StarOutlineIcon v-else :size="24" />
            </template>
          </NcActionButton>
          <NcActionButton
            v-if="!routeIsPublic"
            :aria-label="t('memories', 'Sidebar')"
            @click="toggleSidebar"
            :close-after-click="true"
          >
            {{ t("memories", "Sidebar") }}
            <template #icon>
              <InfoIcon :size="24" />
            </template>
          </NcActionButton>
          <NcActionButton
            v-if="canEdit && !routeIsPublic"
            :aria-label="t('memories', 'Edit')"
            @click="openEditor"
            :close-after-click="true"
          >
            {{ t("memories", "Edit") }}
            <template #icon>
              <TuneIcon :size="24" />
            </template>
          </NcActionButton>
          <NcActionButton
            :aria-label="t('memories', 'Download')"
            @click="downloadCurrent"
            :close-after-click="true"
          >
            {{ t("memories", "Download") }}
            <template #icon>
              <DownloadIcon :size="24" />
            </template>
          </NcActionButton>
          <NcActionButton
            v-if="currentPhoto?.liveid"
            :aria-label="t('memories', 'Download Video')"
            @click="downloadCurrentLiveVideo"
            :close-after-click="true"
          >
            {{ t("memories", "Download Video") }}
            <template #icon>
              <DownloadIcon :size="24" />
            </template>
          </NcActionButton>
          <NcActionButton
            v-if="!routeIsPublic"
            :aria-label="t('memories', 'View in folder')"
            @click="viewInFolder"
            :close-after-click="true"
          >
            {{ t("memories", "View in folder") }}
            <template #icon>
              <OpenInNewIcon :size="24" />
            </template>
          </NcActionButton>
        </NcActions>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Emit, Mixins } from "vue-property-decorator";
import GlobalMixin from "../mixins/GlobalMixin";

import { IDay, IPhoto, IRow, IRowType } from "../types";

import { NcActions, NcActionButton } from "@nextcloud/vue";
import { subscribe, unsubscribe } from "@nextcloud/event-bus";
import { generateUrl } from "@nextcloud/router";
import { showError } from "@nextcloud/dialogs";

import ImageEditor from "./ImageEditor.vue";

import * as dav from "../services/DavRequests";
import * as utils from "../services/Utils";
import { getPreviewUrl } from "../services/FileUtils";
import { getDownloadLink } from "../services/DavRequests";

import PhotoSwipe, { PhotoSwipeOptions } from "photoswipe";
import "photoswipe/style.css";

import PsVideo from "./PsVideo";
import PsLivePhoto from "./PsLivePhoto";

import ShareIcon from "vue-material-design-icons/ShareVariant.vue";
import DeleteIcon from "vue-material-design-icons/TrashCanOutline.vue";
import StarIcon from "vue-material-design-icons/Star.vue";
import StarOutlineIcon from "vue-material-design-icons/StarOutline.vue";
import DownloadIcon from "vue-material-design-icons/Download.vue";
import InfoIcon from "vue-material-design-icons/InformationOutline.vue";
import OpenInNewIcon from "vue-material-design-icons/OpenInNew.vue";
import TuneIcon from "vue-material-design-icons/Tune.vue";

@Component({
  components: {
    NcActions,
    NcActionButton,
    ImageEditor,
    ShareIcon,
    DeleteIcon,
    StarIcon,
    StarOutlineIcon,
    DownloadIcon,
    InfoIcon,
    OpenInNewIcon,
    TuneIcon,
  },
})
export default class Viewer extends Mixins(GlobalMixin) {
  @Emit("deleted") deleted(photos: IPhoto[]) {}
  @Emit("fetchDay") fetchDay(dayId: number) {}
  @Emit("updateLoading") updateLoading(delta: number) {}

  public isOpen = false;
  private originalTitle = null;
  public editorOpen = false;

  private show = false;
  private showControls = false;
  private fullyOpened = false;
  private sidebarOpen = false;
  private sidebarWidth = 400;
  private outerWidth = "100vw";

  /** User interaction detection */
  private activityTimer = 0;

  /** Base dialog */
  private photoswipe: PhotoSwipe | null = null;

  private list: IPhoto[] = [];
  private days = new Map<number, IDay>();
  private dayIds: number[] = [];

  private globalCount = 0;
  private globalAnchor = -1;
  private currIndex = -1;

  mounted() {
    subscribe("files:sidebar:opened", this.handleAppSidebarOpen);
    subscribe("files:sidebar:closed", this.handleAppSidebarClose);
    subscribe("files:file:created", this.handleFileUpdated);
    subscribe("files:file:updated", this.handleFileUpdated);
  }

  beforeDestroy() {
    unsubscribe("files:sidebar:opened", this.handleAppSidebarOpen);
    unsubscribe("files:sidebar:closed", this.handleAppSidebarClose);
    unsubscribe("files:file:created", this.handleFileUpdated);
    unsubscribe("files:file:updated", this.handleFileUpdated);
  }

  /** Number of buttons to show inline */
  get numInlineActions() {
    let base = 3;
    if (this.canShare) base++;
    if (this.canEdit) base++;

    if (globalThis.windowInnerWidth < 768) {
      return Math.min(base, 3);
    } else {
      return Math.min(base, 5);
    }
  }

  /** Route is public */
  get routeIsPublic() {
    return this.$route.name === "folder-share";
  }

  /** Update the document title */
  private updateTitle(photo: IPhoto | undefined) {
    if (!this.originalTitle) {
      this.originalTitle = document.title;
    }
    if (photo) {
      document.title = `${photo.basename} - ${globalThis.OCA.Theming?.name}`;
    } else {
      document.title = this.originalTitle;
      this.originalTitle = null;
    }
  }

  /** Get the currently open photo */
  get currentPhoto() {
    if (!this.list.length || !this.photoswipe) {
      return null;
    }
    const idx = this.currIndex - this.globalAnchor;
    if (idx < 0 || idx >= this.list.length) {
      return null;
    }
    return this.list[idx];
  }

  /** Get download link for current photo */
  get currentDownloadLink() {
    return this.currentPhoto
      ? window.location.origin + getDownloadLink(this.currentPhoto)
      : null;
  }

  /** Event on file changed */
  private handleFileUpdated({ fileid }: { fileid: number }) {
    if (this.currentPhoto && this.currentPhoto.fileid === fileid) {
      this.currentPhoto.etag += "_";
      this.photoswipe.refreshSlideContent(this.currIndex);
    }
  }

  /** User interacted with the page with mouse */
  private setUiVisible(evt: any) {
    clearTimeout(this.activityTimer);
    if (evt) {
      // If directly triggered, always update ui visibility
      // If triggered through a pointer event, only update if this is not
      // a touch event (i.e. a mouse move).
      // On touch devices, tapAction directly handles the ui visibility
      // through Photoswipe.
      const isPointer = evt instanceof PointerEvent;
      const isMouse = isPointer && evt.pointerType !== "touch";
      if (this.isOpen && (!isPointer || isMouse)) {
        this.photoswipe?.template?.classList.add("pswp--ui-visible");

        if (isMouse) {
          this.activityTimer = window.setTimeout(() => {
            if (this.isOpen) {
              this.photoswipe?.template?.classList.remove("pswp--ui-visible");
            }
          }, 2000);
        }
      }
    } else {
      this.photoswipe?.template?.classList.remove("pswp--ui-visible");
    }
  }

  /** Create the base photoswipe object */
  private async createBase(args: PhotoSwipeOptions) {
    this.show = true;
    await this.$nextTick();

    this.photoswipe = new PhotoSwipe({
      counter: true,
      zoom: false,
      loop: false,
      wheelToZoom: true,
      bgOpacity: 1,
      appendToEl: this.$refs.inner as HTMLElement,
      preload: [2, 2],

      easing: "cubic-bezier(.49,.85,.55,1)",
      showHideAnimationType: "zoom",
      showAnimationDuration: 250,
      hideAnimationDuration: 250,

      closeTitle: this.t("memories", "Close"),
      arrowPrevTitle: this.t("memories", "Previous"),
      arrowNextTitle: this.t("memories", "Next"),
      getViewportSizeFn: () => {
        const sidebarWidth = this.sidebarOpen ? this.sidebarWidth : 0;
        this.outerWidth = `calc(100vw - ${sidebarWidth}px)`;
        return {
          x: globalThis.windowInnerWidth - sidebarWidth,
          y: globalThis.windowInnerHeight,
        };
      },
      ...args,
    });

    // Debugging only
    globalThis.photoswipe = this.photoswipe;

    // Monkey patch for focus trapping in sidebar
    const _onFocusIn = this.photoswipe.keyboard._onFocusIn;
    this.photoswipe.keyboard._onFocusIn = (e: FocusEvent) => {
      if (e.target instanceof HTMLElement) {
        if (
          e.target.closest("aside.app-sidebar") ||
          e.target.closest(".v-popper__popper") ||
          e.target.closest(".modal-mask")
        ) {
          return;
        }
      }
      _onFocusIn.call(this.photoswipe.keyboard, e);
    };

    // Refresh sidebar on change
    this.photoswipe.on("change", () => {
      if (this.sidebarOpen) {
        this.openSidebar();
      }
    });

    // Make sure buttons are styled properly
    this.photoswipe.addFilter("uiElement", (element, data) => {
      // add button-vue class if button
      if (element.classList.contains("pswp__button")) {
        element.classList.add("button-vue");
      }
      return element;
    });

    // Total number of photos in this view
    this.photoswipe.addFilter("numItems", (numItems) => {
      return this.globalCount;
    });

    // Put viewer over everything else
    const navElem = document.getElementById("app-navigation-vue");
    const klass = "has-viewer";
    this.photoswipe.on("beforeOpen", () => {
      document.body.classList.add(klass);
      if (navElem) navElem.style.zIndex = "0";
    });
    this.photoswipe.on("openingAnimationStart", () => {
      this.isOpen = true;
      this.fullyOpened = false;
      if (this.sidebarOpen) {
        this.openSidebar();
      }
    });
    this.photoswipe.on("openingAnimationEnd", () => {
      this.fullyOpened = true;
    });
    this.photoswipe.on("close", () => {
      this.isOpen = false;
      this.fullyOpened = false;
      this.setUiVisible(false);
      this.hideSidebar();
      this.setRouteHash(undefined);
      this.updateTitle(undefined);
    });
    this.photoswipe.on("destroy", () => {
      document.body.classList.remove(klass);
      if (navElem) navElem.style.zIndex = "";

      // reset everything
      this.show = false;
      this.isOpen = false;
      this.fullyOpened = false;
      this.photoswipe = null;
      this.list = [];
      this.days.clear();
      this.dayIds = [];
      this.globalCount = 0;
      this.globalAnchor = -1;
    });

    // Update vue route for deep linking
    this.photoswipe.on("slideActivate", (e) => {
      this.currIndex = this.photoswipe.currIndex;
      const photo = e.slide?.data?.photo;
      this.setRouteHash(photo);
      this.updateTitle(photo);
      globalThis.currentViewerPhoto = photo;
    });

    // Show and hide controls
    this.photoswipe.on("uiRegister", (e) => {
      if (this.photoswipe?.template) {
        new MutationObserver((mutations) => {
          mutations.forEach((mutationRecord) => {
            this.showControls = (<HTMLElement>(
              mutationRecord.target
            ))?.classList.contains("pswp--ui-visible");
          });
        }).observe(this.photoswipe.template, {
          attributes: true,
          attributeFilter: ["class"],
        });
      }
    });

    // Video support
    new PsVideo(this.photoswipe, {
      videoAttributes: { controls: "", playsinline: "", preload: "none" },
      autoplay: true,
      preventDragOffset: 40,
    });

    // Live photo support
    new PsLivePhoto(this.photoswipe, {});

    return this.photoswipe;
  }

  /** Open using start photo and rows list */
  public async open(anchorPhoto: IPhoto, rows?: IRow[]) {
    this.list = [...anchorPhoto.d.detail];
    let startIndex = -1;

    // Get days list and map
    for (const r of rows) {
      if (r.type === IRowType.HEAD) {
        if (this.TagDayIDValueSet.has(r.dayId)) continue;

        if (r.day.dayid == anchorPhoto.d.dayid) {
          startIndex = r.day.detail.indexOf(anchorPhoto);
          this.globalAnchor = this.globalCount;
        }

        this.globalCount += r.day.count;
        this.days.set(r.day.dayid, r.day);
        this.dayIds.push(r.day.dayid);
      }
    }

    // Create basic viewer
    await this.createBase({
      index: this.globalAnchor + startIndex,
    });

    // Lazy-generate item data.
    // Load the next two days in the timeline.
    this.photoswipe.addFilter("itemData", (itemData, index) => {
      // Get photo object from list
      let idx = index - this.globalAnchor;
      if (idx < 0) {
        // Load previous day
        const firstDayId = this.list[0].d.dayid;
        const firstDayIdx = utils.binarySearch(this.dayIds, firstDayId);
        if (firstDayIdx === 0) {
          // No previous day
          return {};
        }
        const prevDayId = this.dayIds[firstDayIdx - 1];
        const prevDay = this.days.get(prevDayId);
        if (!prevDay.detail) {
          console.error("[BUG] No detail for previous day");
          return {};
        }
        this.list.unshift(...prevDay.detail);
        this.globalAnchor -= prevDay.count;
      } else if (idx >= this.list.length) {
        // Load next day
        const lastDayId = this.list[this.list.length - 1].d.dayid;
        const lastDayIdx = utils.binarySearch(this.dayIds, lastDayId);
        if (lastDayIdx === this.dayIds.length - 1) {
          // No next day
          return {};
        }
        const nextDayId = this.dayIds[lastDayIdx + 1];
        const nextDay = this.days.get(nextDayId);
        if (!nextDay.detail) {
          console.error("[BUG] No detail for next day");
          return {};
        }
        this.list.push(...nextDay.detail);
      }

      idx = index - this.globalAnchor;
      const photo = this.list[idx];

      // Something went really wrong
      if (!photo) {
        return {};
      }

      // Preload next and previous 3 days
      const dayIdx = utils.binarySearch(this.dayIds, photo.d.dayid);
      const preload = (idx: number) => {
        if (
          idx > 0 &&
          idx < this.dayIds.length &&
          !this.days.get(this.dayIds[idx]).detail
        ) {
          this.fetchDay(this.dayIds[idx]);
        }
      };
      preload(dayIdx - 1);
      preload(dayIdx - 2);
      preload(dayIdx - 3);
      preload(dayIdx + 1);
      preload(dayIdx + 2);
      preload(dayIdx + 3);

      // Get thumb image
      const thumbSrc: string =
        this.thumbElem(photo)?.getAttribute("src") ||
        getPreviewUrl(photo, false, 256);

      // Get full image
      return {
        ...this.getItemData(photo),
        msrc: thumbSrc,
      };
    });

    // Get the thumbnail image
    this.photoswipe.addFilter("thumbEl", (thumbEl, data, index) => {
      const photo = this.list[index - this.globalAnchor];
      if (!photo || !photo.w || !photo.h) return thumbEl;
      return this.thumbElem(photo) || thumbEl;
    });

    // Scroll to keep the thumbnail in view
    this.photoswipe.on("slideActivate", (e) => {
      const thumb = this.thumbElem(e.slide.data?.photo);
      if (thumb && this.fullyOpened) {
        const rect = thumb.getBoundingClientRect();
        if (rect.bottom < 50 || rect.top > globalThis.windowInnerHeight - 50) {
          thumb.scrollIntoView({
            block: "center",
          });
        }
      }
    });

    this.photoswipe.init();
  }

  /** Close the viewer */
  public close() {
    this.photoswipe?.close();
  }

  /** Open with a static list of photos */
  public async openStatic(photo: IPhoto, list: IPhoto[], thumbSize?: number) {
    this.list = list;
    await this.createBase({
      index: list.findIndex((p) => p.fileid === photo.fileid),
    });

    this.globalCount = list.length;
    this.globalAnchor = 0;

    this.photoswipe.addFilter("itemData", (itemData, index) => ({
      ...this.getItemData(this.list[index]),
      msrc: thumbSize ? getPreviewUrl(photo, false, thumbSize) : undefined,
    }));

    this.isOpen = true;
    this.photoswipe.init();
  }

  /** Get base data object */
  private getItemData(photo: IPhoto) {
    let previewUrl = getPreviewUrl(photo, false, 1024);
    const isvideo = photo.flag & this.c.FLAG_IS_VIDEO;

    // Preview aren't animated
    if (photo.mimetype === "image/gif") {
      previewUrl = getDownloadLink(photo);
    } else if (isvideo) {
      previewUrl = generateUrl(getDownloadLink(photo));
    }

    // Get height and width
    let w = photo.w;
    let h = photo.h;

    if (isvideo && w && h) {
      // For videos, make sure the screen is filled up,
      // by scaling up the video by a maximum of 4x
      w *= 4;
      h *= 4;
    }

    return {
      src: previewUrl,
      width: w || undefined,
      height: h || undefined,
      thumbCropped: true,
      photo: photo,
      type: isvideo ? "video" : "image",
    };
  }

  /** Get element for thumbnail if it exists */
  private thumbElem(photo: IPhoto): HTMLImageElement | undefined {
    if (!photo) return;
    const elems = document.querySelectorAll(`.memories-thumb-${photo.key}`);

    if (elems.length === 0) return;
    if (elems.length === 1) return elems[0] as HTMLImageElement;

    // Find element within 500px of the screen top
    let elem: HTMLImageElement;
    elems.forEach((e) => {
      const rect = e.getBoundingClientRect();
      if (rect.top > -500) {
        elem = e as HTMLImageElement;
      }
    });

    return elem;
  }

  /** Set the route hash to the given photo */
  private setRouteHash(photo: IPhoto | undefined) {
    if (!photo) {
      if (!this.isOpen && this.$route.hash?.startsWith("#v")) {
        this.$router.back();

        // Ensure this does not have the hash, otherwise replace it
        if (this.$route.hash?.startsWith("#v")) {
          this.$router.replace({
            hash: "",
          });
        }
      }
      return;
    }
    const hash = photo ? utils.getViewerHash(photo) : "";
    const route = {
      ...this.$route,
      hash,
    };
    if (hash !== this.$route.hash) {
      if (this.$route.hash) {
        this.$router.replace(route);
      } else {
        this.$router.push(route);
      }
    }
  }

  get canEdit() {
    return (
      this.currentPhoto?.mimetype?.startsWith("image/") &&
      !this.currentPhoto.liveid
    );
  }

  private openEditor() {
    // Only for JPEG for now
    if (!this.canEdit) return;
    this.editorOpen = true;
  }

  /** Does the browser support native share API */
  get canShare() {
    return (
      "share" in navigator &&
      this.currentPhoto &&
      !(this.currentPhoto.flag & this.c.FLAG_IS_VIDEO)
    );
  }

  /** Share the current photo externally */
  private async shareCurrent() {
    try {
      // Check navigator support
      if (!this.canShare) throw new Error("Share not supported");

      // Get image data from "img.pswp__img"
      const img = document.querySelector("img.pswp__img") as HTMLImageElement;
      if (!img?.src) return;

      // Shre image data using navigator api
      const photo = this.currentPhoto;
      if (!photo) return;

      // No videos yet
      if (photo.flag & this.c.FLAG_IS_VIDEO)
        throw new Error(this.t("memories", "Video sharing not supported yet"));

      // Get image blob
      const blob = await (await fetch(img.src)).blob();

      // Fix basename extension
      let basename = photo.basename;
      let targetExts = [];
      if (photo.mimetype === "image/png") {
        targetExts = ["png"];
      } else {
        targetExts = ["jpg", "jpeg"];
      }

      // Append extension if not found
      if (!targetExts.includes(basename.split(".").pop().toLowerCase())) {
        basename += "." + targetExts[0];
      }

      const data = {
        files: [
          new File([blob], basename, {
            type: blob.type,
          }),
        ],
        title: photo.basename,
        text: photo.basename,
      };

      if (!(<any>navigator).canShare(data)) {
        throw new Error(this.t("memories", "Cannot share this type of data"));
      }

      try {
        await navigator.share(data);
      } catch (e) {
        // Don't show this error because it's silly stuff
        // like "share canceled"
        console.error(e);
      }
    } catch (err) {
      console.error(err.name, err.message);
      showError(err.message);
    }
  }

  /** Delete this photo and refresh */
  private async deleteCurrent() {
    const idx = this.photoswipe.currIndex - this.globalAnchor;

    // Delete with WebDAV
    try {
      this.updateLoading(1);
      for await (const p of dav.deletePhotos([this.list[idx]])) {
        if (!p[0]) return;
      }
    } finally {
      this.updateLoading(-1);
    }

    const spliced = this.list.splice(idx, 1);
    this.globalCount--;
    for (let i = idx - 3; i <= idx + 3; i++) {
      this.photoswipe.refreshSlideContent(i + this.globalAnchor);
    }
    this.deleted(spliced);
  }

  /** Is the current photo a favorite */
  private isFavorite() {
    const p = this.currentPhoto;
    if (!p) return false;
    return Boolean(p.flag & this.c.FLAG_IS_FAVORITE);
  }

  /** Favorite the current photo */
  private async favoriteCurrent() {
    const photo = this.currentPhoto;
    const val = !this.isFavorite();
    try {
      this.updateLoading(1);
      for await (const p of dav.favoritePhotos([photo], val)) {
        if (!p[0]) return;
        this.$forceUpdate();
      }
    } finally {
      this.updateLoading(-1);
    }

    // Set flag on success
    if (val) {
      photo.flag |= this.c.FLAG_IS_FAVORITE;
    } else {
      photo.flag &= ~this.c.FLAG_IS_FAVORITE;
    }
  }

  /** Download the current photo */
  private async downloadCurrent() {
    const photo = this.currentPhoto;
    if (!photo) return;
    dav.downloadFilesByPhotos([photo]);
  }

  /** Download live part of current video */
  private async downloadCurrentLiveVideo() {
    const photo = this.currentPhoto;
    if (!photo) return;
    window.location.href = utils.getLivePhotoVideoUrl(photo);
  }

  /** Open the sidebar */
  private async openSidebar(photo?: IPhoto) {
    const fInfo = await dav.getFiles([photo || this.currentPhoto]);
    globalThis.OCA?.Files?.Sidebar?.setFullScreenMode?.(true);
    globalThis.OCA.Files.Sidebar.setActiveTab("memories-metadata");
    globalThis.OCA.Files.Sidebar.open(fInfo[0].filename);
  }

  private async updateSizeWithoutAnim() {
    const wasFullyOpened = this.fullyOpened;
    this.fullyOpened = false;
    this.photoswipe.updateSize();
    await new Promise((resolve) => setTimeout(resolve, 200));
    this.fullyOpened = wasFullyOpened;
  }

  private handleAppSidebarOpen() {
    if (this.show && this.photoswipe) {
      const sidebar: HTMLElement = document.querySelector("aside.app-sidebar");
      if (sidebar) {
        this.sidebarWidth = sidebar.offsetWidth - 2;
      }

      this.sidebarOpen = true;
      this.updateSizeWithoutAnim();
    }
  }

  private handleAppSidebarClose() {
    if (this.show && this.photoswipe && this.fullyOpened) {
      this.sidebarOpen = false;
      this.updateSizeWithoutAnim();
    }
  }

  /** Hide the sidebar, without marking it as closed */
  private hideSidebar() {
    globalThis.OCA?.Files?.Sidebar?.close();
    globalThis.OCA?.Files?.Sidebar?.setFullScreenMode?.(false);
  }

  /** Close the sidebar */
  private closeSidebar() {
    this.hideSidebar();
    this.sidebarOpen = false;
    this.photoswipe.updateSize();
  }

  /** Toggle the sidebar visibility */
  private toggleSidebar() {
    if (this.sidebarOpen) {
      this.closeSidebar();
    } else {
      this.openSidebar();
    }
  }

  /**
   * Open the files app with the current file.
   */
  private async viewInFolder() {
    if (this.currentPhoto) dav.viewInFolder(this.currentPhoto);
  }
}
</script>

<style lang="scss" scoped>
.outer {
  z-index: 3000;
  width: 100vw;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
  overflow: hidden;
  color: white;
}

.top-bar {
  z-index: 100001;
  position: absolute;
  top: 8px;
  right: 50px;

  :deep .button-vue--icon-only {
    color: white;
    background-color: transparent !important;
  }

  transition: opacity 0.2s ease-in-out;
  opacity: 0;
  &.showControls {
    opacity: 1;
  }
}

.fullyOpened :deep .pswp__container {
  @media (min-width: 1024px) {
    // Animate transitions
    // Disabled because this makes you sick if moving fast
    // transition: transform var(--pswp-transition-duration) ease !important;
  }
}

.inner,
.inner :deep .pswp {
  width: inherit;

  .pswp__top-bar {
    background: linear-gradient(0deg, transparent, rgba(0, 0, 0, 0.3));
  }
}

:deep .video-js .vjs-big-play-button {
  display: none;
}

:deep .plyr__volume {
  // Cannot be vertical yet :(
  @media (max-width: 768px) {
    display: none;
  }
}

:deep .pswp {
  contain: strict;

  .pswp__zoom-wrap {
    width: 100%;
  }

  img.pswp__img {
    object-fit: contain;
  }

  .pswp__button {
    color: white;

    &,
    * {
      cursor: pointer;
    }
  }
  .pswp__icn-shadow {
    display: none;
  }

  // Hide arrows on mobile
  @media (max-width: 768px) {
    .pswp__button--arrow {
      opacity: 0 !important;
    }
  }
}
</style>