<script setup lang="ts">
import {
  Primitive,
  usePrimitiveElement,
  type PrimitiveProps,
} from "@/Primitive";
import {
  focus,
  focusFirst,
  getTabbableCandidates,
  getTabbableEdges,
  AUTOFOCUS_ON_MOUNT,
  AUTOFOCUS_ON_UNMOUNT,
  EVENT_OPTIONS,
} from "./utils";
import { removeLinks } from "./stack";
import { nextTick, ref, watchEffect } from "vue";

export interface FocusScopeProps extends PrimitiveProps {
  /**
   * When `true`, tabbing from last item will focus first tabbable
   * and shift+tab from first item will focus last tababble.
   * @defaultValue false
   */
  loop?: boolean;

  /**
   * When `true`, focus cannot escape the focus scope via keyboard,
   * pointer, or a programmatic focus.
   * @defaultValue false
   */
  trapped?: boolean;
}

export interface FocusScopeEmits {
  /**
   * Event handler called when auto-focusing on mount.
   * Can be prevented.
   */
  (e: "mountAutoFocus", event: Event): void;

  /**
   * Event handler called when auto-focusing on unmount.
   * Can be prevented.
   */
  (e: "unmountAutoFocus", event: Event): void;
}

const props = withDefaults(defineProps<FocusScopeProps>(), {
  loop: false,
  trapped: false,
});
const emits = defineEmits<FocusScopeEmits>();

const { primitiveElement, currentElement } = usePrimitiveElement();
const lastFocusedElementRef = ref<HTMLElement | null>(null);

watchEffect((cleanupFn) => {
  const container = currentElement.value;
  if (!props.trapped) return;

  function handleFocusIn(event: FocusEvent) {
    if (!container) return;
    const target = event.target as HTMLElement | null;
    if (container.contains(target)) {
      lastFocusedElementRef.value = target;
    } else {
      focus(lastFocusedElementRef.value, { select: true });
    }
  }

  function handleFocusOut(event: FocusEvent) {
    if (!container) return;
    const relatedTarget = event.relatedTarget as HTMLElement | null;

    // A `focusout` event with a `null` `relatedTarget` will happen in at least two cases:
    //
    // 1. When the user switches app/tabs/windows/the browser itself loses focus.
    // 2. In Google Chrome, when the focused element is removed from the DOM.
    //
    // We let the browser do its thing here because:
    //
    // 1. The browser already keeps a memory of what's focused for when the page gets refocused.
    // 2. In Google Chrome, if we try to focus the deleted focused element (as per below), it
    //    throws the CPU to 100%, so we avoid doing anything for this reason here too.
    if (relatedTarget === null) return;

    // If the focus has moved to an actual legitimate element (`relatedTarget !== null`)
    // that is outside the container, we move focus to the last valid focused element inside.
    if (!container.contains(relatedTarget)) {
      focus(lastFocusedElementRef.value, { select: true });
    }
  }

  // When the focused element gets removed from the DOM, browsers move focus
  // back to the document.body. In this case, we move focus to the container
  // to keep focus trapped correctly.
  function handleMutations(mutations: MutationRecord[]) {
    const focusedElement = document.activeElement as HTMLElement | null;
    if (focusedElement !== document.body) return;
    for (const mutation of mutations) {
      if (mutation.removedNodes.length > 0) focus(container);
    }
  }

  document.addEventListener("focusin", handleFocusIn);
  document.addEventListener("focusout", handleFocusOut);
  const mutationObserver = new MutationObserver(handleMutations);
  if (container)
    mutationObserver.observe(container, { childList: true, subtree: true });

  cleanupFn(() => {
    document.removeEventListener("focusin", handleFocusIn);
    document.removeEventListener("focusout", handleFocusOut);
    mutationObserver.disconnect();
  });
});

watchEffect(async (cleanupFn) => {
  const container = currentElement.value;

  await nextTick();
  if (!container) return;

  const previouslyFocusedElement = document.activeElement as HTMLElement | null;
  const hasFocusedCandidate = container.contains(previouslyFocusedElement);

  if (!hasFocusedCandidate) {
    const mountEvent = new CustomEvent(AUTOFOCUS_ON_MOUNT, EVENT_OPTIONS);
    container.addEventListener(AUTOFOCUS_ON_MOUNT, (ev) =>
      emits("mountAutoFocus", ev)
    );
    container.dispatchEvent(mountEvent);

    if (!mountEvent.defaultPrevented) {
      focusFirst(removeLinks(getTabbableCandidates(container)), {
        select: true,
      });
      if (document.activeElement === previouslyFocusedElement) {
        focus(container);
      }
    }
  }

  cleanupFn(() => {
    container.removeEventListener(AUTOFOCUS_ON_MOUNT, (ev) =>
      emits("mountAutoFocus", ev)
    );

    setTimeout(() => {
      const unmountEvent = new CustomEvent(AUTOFOCUS_ON_UNMOUNT, EVENT_OPTIONS);
      container.addEventListener(AUTOFOCUS_ON_UNMOUNT, (ev) =>
        emits("unmountAutoFocus", ev)
      );
      container.dispatchEvent(unmountEvent);
      if (!unmountEvent.defaultPrevented) {
        focus(previouslyFocusedElement ?? document.body, { select: true });
      }
      // we need to remove the listener after we `dispatchEvent`
      container.removeEventListener(AUTOFOCUS_ON_UNMOUNT, (ev) =>
        emits("unmountAutoFocus", ev)
      );
    }, 0);
  });
});

const handleKeyDown = (event: KeyboardEvent) => {
  if (!props.loop && !props.trapped) return;

  const isTabKey =
    event.key === "Tab" && !event.altKey && !event.ctrlKey && !event.metaKey;
  const focusedElement = document.activeElement as HTMLElement | null;

  if (isTabKey && focusedElement) {
    const container = event.currentTarget as HTMLElement;
    const [first, last] = getTabbableEdges(container);
    const hasTabbableElementsInside = first && last;

    // we can only wrap focus if we have tabbable edges
    if (!hasTabbableElementsInside) {
      if (focusedElement === container) event.preventDefault();
    } else {
      if (!event.shiftKey && focusedElement === last) {
        event.preventDefault();
        if (props.loop) focus(first, { select: true });
      } else if (event.shiftKey && focusedElement === first) {
        event.preventDefault();
        if (props.loop) focus(last, { select: true });
      }
    }
  }
};
</script>

<template>
  <Primitive
    ref="primitiveElement"
    tabIndex="-1"
    @keydown="handleKeyDown"
    :asChild="props.asChild"
    :as="props.as"
  >
    <slot></slot>
  </Primitive>
</template>