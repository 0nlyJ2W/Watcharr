<!-- To be used from Rating component -->

<script lang="ts">
  import Icon from "../Icon.svelte";
  import tooltip from "../actions/tooltip";

  export let rating: number | undefined;
  export let onChange: (newRating: number) => Promise<boolean>;

  $: r = rating ? Math.round(rating) : 0;
</script>

<div class="thumbs-ctr">
  <button
    use:tooltip={{ text: "Disliked", pos: "top" }}
    on:click={() => onChange(1)}
    class={r && r > 0 && r < 5 ? "active" : ""}
  >
    <Icon i="thumb-down" />
  </button>
  <button
    use:tooltip={{ text: "Mediocre", pos: "top" }}
    on:click={() => onChange(5)}
    class={r && r > 4 && r < 8 ? "active" : ""}
  >
    <span>-</span>
  </button>
  <button
    use:tooltip={{ text: "Liked", pos: "top" }}
    on:click={() => onChange(9)}
    class={r && r > 7 ? "active" : ""}
  >
    <Icon i="thumb-up" />
  </button>
</div>

<style lang="scss">
  .thumbs-ctr {
    display: flex;
    flex-flow: row;
    gap: 10px;

    button {
      display: flex;
      align-items: center;
      justify-content: center;
      height: 60px;

      &:nth-child(2) span {
        /* Center the dash (-) for the 'mediocre' rating button */
        transform: translateY(5px);
        font-size: 50px;
        font-family: "Shrikhand";
      }

      &.active {
        color: gold;
        fill: gold;
        background-color: $text-color;
      }
    }
  }
</style>
