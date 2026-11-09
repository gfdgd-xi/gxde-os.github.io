<script setup lang="ts">
import DefaultTheme from "vitepress/theme";
import { nextTick, provide } from "vue";
import { useData } from "vitepress";

const { isDark } = useData();

const { Layout } = DefaultTheme;

let isTransitioning = false;

const enableTransitions = () =>
  "startViewTransition" in document &&
  window.matchMedia("(prefers-reduced-motion: no-preference)").matches;

provide("toggle-appearance", async (event: MouseEvent) => {
  if (isTransitioning) return;

  if (!enableTransitions()) {
    isDark.value = !isDark.value;
    return;
  }

  const { clientX: x, clientY: y } = event;
  const toggle = event.currentTarget as HTMLElement;
  const root = document.documentElement;
  const clipPath = [
    `circle(0px at ${x}px ${y}px)`,
    `circle(${Math.hypot(
      Math.max(x, innerWidth - x),
      Math.max(y, innerHeight - y),
    )}px at ${x}px ${y}px)`,
  ];

  isTransitioning = true;
  toggle.classList.add("appearance-transitioning");
  toggle.classList.toggle("appearance-hovered", toggle.matches(":hover"));
  // Mask the light snapshot before its JavaScript animation is ready.
  root.style.setProperty("--appearance-transition-clip", clipPath[0]);
  let appearanceAnimation: Animation | undefined;

  try {
    const transition = document.startViewTransition(async () => {
      isDark.value = !isDark.value;
      await nextTick();
    });

    await transition.ready.then(
      () => {
        appearanceAnimation = root.animate(
          { clipPath: isDark.value ? clipPath.reverse() : clipPath },
          {
            duration: 300,
            easing: "ease-in",
            fill: "both",
            pseudoElement: `::view-transition-${isDark.value ? "old" : "new"}(root)`,
          },
        );
      },
      () => {
        // Skipped transitions still apply the theme change.
      },
    );

    await transition.finished;
  } finally {
    // Allow pointer hit-testing to resume before restoring hover transitions.
    await new Promise<void>((resolve) => {
      requestAnimationFrame(() => requestAnimationFrame(() => resolve()));
    });
    // A filled animation would otherwise affect the next transition snapshot.
    appearanceAnimation?.cancel();
    root.style.removeProperty("--appearance-transition-clip");
    toggle.classList.remove("appearance-transitioning", "appearance-hovered");
    isTransitioning = false;
  }
});
</script>

<template>
  <Layout>
    <template #nav-bar-content-after>
      <a href="https://github.com/GXDE-OS" style="padding: 0 12px">
        <img src="https://img.shields.io/github/stars/GXDE-OS?style=flat&logo=github" alt="star" />
      </a>
    </template>
  </Layout>
</template>

<style>
::view-transition-old(root),
::view-transition-new(root) {
  animation: none;
  mix-blend-mode: normal;
}

::view-transition-old(root),
.dark::view-transition-new(root) {
  z-index: 1;
}

::view-transition-new(root),
.dark::view-transition-old(root) {
  z-index: 9999;
}

::view-transition-new(root) {
  clip-path: var(--appearance-transition-clip, none);
}

.dark::view-transition-new(root) {
  clip-path: none;
}

.VPSwitchAppearance {
  width: 22px !important;
}

.VPSwitchAppearance .check {
  transform: none !important;
}

.VPSwitchAppearance.appearance-transitioning,
.VPSwitchAppearance.appearance-transitioning .check .icon span {
  transition: none !important;
}

.VPSwitchAppearance.appearance-hovered {
  border-color: var(--vp-c-brand-1) !important;
}
</style>
