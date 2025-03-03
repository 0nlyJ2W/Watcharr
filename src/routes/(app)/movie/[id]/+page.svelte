<script lang="ts">
  import PageError from "@/lib/PageError.svelte";
  import PersonPoster from "@/lib/poster/PersonPoster.svelte";
  import Rating from "@/lib/rating/Rating.svelte";
  import Spinner from "@/lib/Spinner.svelte";
  import Status from "@/lib/Status.svelte";
  import HorizontalList from "@/lib/HorizontalList.svelte";
  import { contentExistsOnJellyfin, removeWatched, updateWatched } from "@/lib/util/api";
  import { serverFeatures, userSettings, watchedList } from "@/store";
  import type {
    ArrDetailsResponse,
    TMDBContentCredits,
    TMDBContentCreditsCrew,
    TMDBMovieDetails,
    WatchedStatus
  } from "@/types";
  import axios from "axios";
  import { getTopCrew } from "@/lib/util/helpers.js";
  import Activity from "@/lib/Activity.svelte";
  import Title from "@/lib/content/Title.svelte";
  import VideoEmbedModal from "@/lib/content/VideoEmbedModal.svelte";
  import ProvidersList from "@/lib/content/ProvidersList.svelte";
  import Icon from "@/lib/Icon.svelte";
  import SimilarContent from "@/lib/content/SimilarContent.svelte";
  import { onMount } from "svelte";
  import { page } from "$app/stores";
  import RequestMovie from "@/lib/request/RequestMovie.svelte";
  import Error from "@/lib/Error.svelte";
  import FollowedThoughts from "@/lib/content/FollowedThoughts.svelte";
  import ArrRequestButton from "@/lib/request/ArrRequestButton.svelte";
  import tooltip from "@/lib/actions/tooltip.js";
  import MyThoughts from "@/lib/content/MyThoughts.svelte";
  import AddToTagButton from "@/lib/tag/AddToTagButton.svelte";

  $: settings = $userSettings;

  export let data;

  let trailer: string | undefined;
  let requestModalShown = false;
  let trailerShown = false;
  let jellyfinUrl: string | undefined;
  let arrRequestButtonComp: ArrRequestButton;

  $: wListItem = $watchedList.find(
    (w) => w.content?.type === "movie" && w.content?.tmdbId === data.movieId
  );

  let movieId: number | undefined;
  let movie: TMDBMovieDetails | undefined;
  let pageError: Error | undefined;
  let arrStatus: ArrDetailsResponse | undefined;

  onMount(() => {
    const unsubscribe = page.subscribe((value) => {
      console.log(value);
      const params = value.params;
      if (params && params.id) {
        movieId = Number(params.id);
      }
    });

    return unsubscribe;
  });

  $: {
    (async () => {
      try {
        movie = undefined;
        pageError = undefined;
        if (!movieId) {
          return;
        }
        const data = (
          await axios.get(`/content/movie/${movieId}`, { params: { region: settings?.country } })
        ).data as TMDBMovieDetails;
        if (data.videos?.results?.length > 0) {
          const t = data.videos.results.find((v) => v.type?.toLowerCase() === "trailer");
          if (t?.key) {
            if (t?.site?.toLowerCase() === "youtube") {
              trailer = `https://www.youtube.com/embed/${t?.key}`;
            }
          }
        }
        contentExistsOnJellyfin("movie", data.title, data.id).then((j) => {
          if (j?.hasContent && j?.url !== "") {
            jellyfinUrl = j.url;
          }
        });
        movie = data;
      } catch (err: any) {
        movie = undefined;
        pageError = err;
      }
    })();
  }

  async function getMovieCredits() {
    const credits = (await axios.get(`/content/movie/${data.movieId}/credits`))
      .data as TMDBContentCredits & { topCrew: TMDBContentCreditsCrew[] };
    if (credits.crew?.length > 0) {
      credits.topCrew = getTopCrew(credits.crew);
    }
    return credits;
  }

  async function contentChanged(
    newStatus?: WatchedStatus,
    newRating?: number,
    newThoughts?: string,
    pinned?: boolean
  ): Promise<boolean> {
    if (!data.movieId) {
      console.error("contentChanged: no movieId");
      return false;
    }
    return await updateWatched(data.movieId, "movie", newStatus, newRating, newThoughts, pinned);
  }
</script>

<svelte:head>
  <title>{movie?.title ? `${movie.title} - ` : ""}Movie</title>
</svelte:head>

{#if pageError}
  <PageError pretty="Failed to load movie!" error={pageError} />
{:else if !movie}
  <Spinner />
{:else if Object.keys(movie).length > 0}
  <div>
    <div class="content">
      {#if movie?.backdrop_path}
        <img
          class="backdrop"
          src={"https://www.themoviedb.org/t/p/w1920_and_h800_multi_faces" + movie.backdrop_path}
          alt=""
        />
      {/if}
      <div class="vignette" />

      <div class="details-container">
        <img class="poster" src={"https://image.tmdb.org/t/p/w500" + movie.poster_path} alt="" />

        <div class="details">
          <Title
            title={movie.title}
            homepage={movie.homepage}
            releaseYear={new Date(Date.parse(movie.release_date)).getFullYear()}
            voteAverage={movie.vote_average}
            voteCount={movie.vote_count}
          />

          <span class="quick-info">
            <span>{movie.runtime} min</span>

            <div>
              {#each movie.genres as g, i}
                <span>{g.name}{i !== movie.genres.length - 1 ? ", " : ""}</span>
              {/each}
            </div>
          </span>

          <!-- <span>{movie.tagline}</span> -->

          <!-- {movie.status} -->

          <span style="font-weight: bold; font-size: 14px;">Overview</span>
          <p>{movie.overview}</p>

          <div class="btns">
            {#if trailer}
              <button on:click={() => (trailerShown = !trailerShown)}>View Trailer</button>
              {#if trailerShown}
                <VideoEmbedModal embed={trailer} closed={() => (trailerShown = false)} />
              {/if}
            {/if}
            {#if jellyfinUrl}
              <a class="btn" href={jellyfinUrl} target="_blank">
                <Icon i="jellyfin" wh={14} />Play On Jellyfin
              </a>
            {/if}
            {#if $serverFeatures.radarr && data.movieId}
              <ArrRequestButton
                type="movie"
                tmdbId={data.movieId}
                openRequestModal={() => (requestModalShown = !requestModalShown)}
                bind:this={arrRequestButtonComp}
              />
            {/if}
            {#if wListItem}
              <div class="other-side">
                <AddToTagButton watchedItem={wListItem} />
                <button
                  on:click={() => {
                    if (wListItem?.pinned) {
                      contentChanged(undefined, undefined, undefined, false);
                    } else {
                      contentChanged(undefined, undefined, undefined, true);
                    }
                  }}
                  use:tooltip={{
                    text: `${wListItem?.pinned ? "Unpin from" : "Pin to"} top of list`,
                    pos: "bot"
                  }}
                >
                  <Icon i={wListItem?.pinned ? "unpin" : "pin"} wh={19} />
                </button>
                <button
                  class="delete-btn"
                  on:click={() =>
                    wListItem
                      ? removeWatched(wListItem.id)
                      : console.error("no wlistItem.. can't delete")}
                  use:tooltip={{ text: "Delete", pos: "bot" }}
                >
                  <Icon i="trash" wh={19} />
                </button>
              </div>
            {/if}
          </div>

          <ProvidersList providers={movie["watch/providers"]} />
        </div>
      </div>
    </div>

    {#if requestModalShown}
      <RequestMovie
        content={movie}
        onClose={(reqResp) => {
          requestModalShown = false;
          if (reqResp) {
            arrRequestButtonComp.setExistingRequest(reqResp);
          }
        }}
      />
    {/if}

    <div class="page">
      <div class="review">
        <!-- <span>What did you think?</span> -->
        <Rating rating={wListItem?.rating} onChange={(n) => contentChanged(undefined, n)} />
        <Status status={wListItem?.status} onChange={(n) => contentChanged(n)} />
        {#if wListItem}
          <MyThoughts
            contentTitle={movie.title}
            thoughts={wListItem?.thoughts}
            onChange={(newThoughts) => {
              return contentChanged(undefined, undefined, newThoughts);
            }}
          />
        {/if}
      </div>

      {#if movieId}
        <FollowedThoughts mediaType="movie" mediaId={movieId} />
      {/if}

      {#await getMovieCredits()}
        <Spinner />
      {:then credits}
        {#if credits.topCrew?.length > 0}
          <div class="creators">
            {#each credits.topCrew as crew}
              <div>
                <span>{crew.name}</span>
                <span>{crew.job}</span>
              </div>
            {/each}
          </div>
        {/if}

        {#if credits.cast?.length > 0}
          <HorizontalList title="Cast">
            {#each credits.cast?.slice(0, 50) as cast}
              <PersonPoster
                id={cast.id}
                name={cast.name}
                path={cast.profile_path}
                role={cast.character}
                zoomOnHover={false}
              />
            {/each}
          </HorizontalList>
        {/if}
      {:catch err}
        <Error error={err} pretty="Failed to load cast!" />
      {/await}

      <SimilarContent type="movie" similar={movie.similar} />

      {#if wListItem}
        <Activity wListId={wListItem.id} activity={wListItem.activity} />
      {/if}
    </div>
  </div>
{:else}
  Movie not found
{/if}

<style lang="scss">
  .content {
    position: relative;
    color: white;

    img.backdrop {
      position: absolute;
      left: 0;
      top: 0;
      z-index: -2;
      width: 100%;
      height: 100%;
      object-fit: cover;
      filter: $backdrop-filter;
      mix-blend-mode: $backdrop-mix-blend-mode;
      mask-image: $backdrop-mask-image;
    }

    .vignette {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background-color: rgba($color: #000000, $alpha: 0.7);
      z-index: -1;
      mask-image: $backdrop-mask-image;
    }

    .details-container {
      display: flex;
      flex-flow: row;
      gap: 35px;
      max-width: 1100px;
      padding: 40px 80px;
      margin-left: auto;
      margin-right: auto;

      img.poster {
        width: 235px;
        height: 100%;
        box-shadow: 0px 0px 14px -4px #9c8080;
        border-radius: 12px;
      }

      .details {
        display: flex;
        flex-flow: column;
        gap: 5px;

        .quick-info {
          display: flex;
          gap: 10px;
          margin-bottom: 8px;
        }

        p {
          font-size: 14px;
          margin-bottom: 18px;
        }

        .btns {
          display: flex;
          flex-flow: row;
          flex-wrap: wrap;
          gap: 8px;
          margin-top: auto;

          a.btn,
          button {
            max-width: fit-content;
            overflow: hidden;
            animation: 50ms cubic-bezier(0.86, 0, 0.07, 1) forwards otherbtn;
            white-space: nowrap;
            gap: 6px;
            justify-content: flex-start;
            font-size: 14px;

            @keyframes otherbtn {
              from {
                width: 0px;
              }
              to {
                width: 100%;
              }
            }
          }

          .other-side {
            display: flex;
            flex-flow: row;
            gap: 8px;

            @media screen and (min-width: 900px) {
              margin-left: auto;
            }
          }

          .delete-btn {
            &:hover {
              color: $error;
            }
          }
        }
      }

      @media screen and (max-width: 700px) {
        padding: 40px;
      }

      @media screen and (max-width: 590px) {
        flex-flow: column;
        align-items: center;
      }
    }
  }

  .page {
    display: flex;
    flex-flow: column;
    align-items: center;
    margin-left: auto;
    margin-right: auto;
    gap: 30px;
    padding: 20px 50px;
    max-width: 1200px;

    @media screen and (max-width: 500px) {
      padding: 20px;
    }
  }

  .review {
    display: flex;
    flex-flow: column;
    gap: 10px;
    width: 100%;
    max-width: 380px;

    @media screen and (max-width: 420px) {
      max-width: 340px;
    }
  }

  .creators {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 35px;
    margin: 10px 60px;

    div {
      display: flex;
      flex-flow: column;
      min-width: 150px;

      span:first-child {
        font-weight: bold;
      }
    }
  }
</style>
