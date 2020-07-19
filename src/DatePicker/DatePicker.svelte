<script>
  export let appendTo = document.body;
  export let dateFormat = "m/d/Y";
  export let datePickerType = "simple";
  export let id = "ccs-" + Math.random().toString(36);
  export let light = false;
  export let locale = "en";
  export let maxDate = null;
  export let minDate = null;
  export let short = false;
  export let value = "";

  import {
    createEventDispatcher,
    setContext,
    afterUpdate,
    onDestroy
  } from "svelte";
  import { writable, derived } from "svelte/store";
  import { createCalendar } from "./createCalendar";

  const dispatch = createEventDispatcher();
  const inputs = writable([]);
  const inputIds = derived(inputs, _ => _.map(({ id }) => id));
  const labelTextEmpty = derived(
    inputs,
    _ => _.filter(({ labelText }) => !!labelText).length === 0
  );
  const inputValue = writable(value);
  const mode = writable(datePickerType);
  const range = derived(mode, _ => _ === "range");
  const hasCalendar = derived(mode, _ => _ === "single" || _ === "range");

  let calendar = undefined;
  let datePickerRef = undefined;
  let inputRef = undefined;
  let inputRefTo = undefined;

  setContext("DatePicker", {
    range,
    inputValue,
    hasCalendar,
    add: data => {
      inputs.update(_ => [..._, data]);
    },
    declareRef: ({ id, ref }) => {
      if ($inputIds.indexOf(id) === 0) {
        inputRef = ref;
      } else {
        inputRefTo = ref;
      }
    },
    updateValue: ({ type, value }) => {
      if ((!calendar && type === "input") || type === "change") {
        inputValue.set(value);
      }

      if (!calendar && type === "change") {
        dispatch("change", value);
      }
    },
    blurInput: relatedTarget => {
      if (calendar && !calendar.calendarContainer.contains(relatedTarget)) {
        calendar.close();
      }
    },
    openCalendar: () => {
      calendar.open();
    },
    focusCalendar: () => {
      (
        calendar.selectedDateElem ||
        calendar.todayDateElem ||
        calendar.calendarContainer.querySelector(".flatpickr-day[tabindex]") ||
        calendar.calendarContainer
      ).focus();
    }
  });

  afterUpdate(() => {
    if ($hasCalendar && !calendar) {
      calendar = createCalendar({
        options: {
          appendTo,
          dateFormat,
          defaultDate: $inputValue,
          locale,
          maxDate,
          minDate,
          mode: $mode
        },
        base: inputRef,
        input: inputRefTo,
        dispatch: event => {
          const detail = { selectedDates: calendar.selectedDates };

          if ($range) {
            detail.dateStr = {
              from: inputRef.value,
              to: inputRefTo.value
            };
          } else {
            detail.dateStr = inputRef.value;
          }

          return dispatch(event, detail);
        }
      });
    }

    if (calendar && !$range) {
      calendar.setDate($inputValue);
    }
  });

  onDestroy(() => {
    if (calendar) {
      calendar.destroy();
    }
  });

  $: inputValue.set(value);
  $: value = $inputValue;
</script>

<svelte:body
  on:click={({ target }) => {
    if (!calendar || !calendar.isOpen) {
      return;
    }
    if (datePickerRef && datePickerRef.contains(target)) {
      return;
    }
    if (!calendar.calendarContainer.contains(target)) {
      calendar.close();
    }
  }} />

<div
  class:bx--form-item={true}
  {...$$restProps}
  on:click
  on:mouseover
  on:mouseenter
  on:mouseleave>
  <div
    bind:this={datePickerRef}
    {id}
    class:bx--date-picker={true}
    class:bx--date-picker--short={short}
    class:bx--date-picker--light={light}
    class="{datePickerType && `--date-picker--${datePickerType}`}
    {datePickerType === 'range' && $labelTextEmpty && '--date-picker--nolabel'}">
    <slot />
  </div>
</div>