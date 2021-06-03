<template>
  <div style="margin: 0 auto; max-width: 1024px; width: 100%">
    <h1 style="margin-bottom: 0.25em">Spotify to Deezer</h1>
    <p style="margin-top: 0; margin-bottom: 2rem">
      Transform a Spotify playlist into deezer widgets
    </p>

    <input
      style="font-size: 1.5rem"
      type="text"
      v-model="id"
      placeholder="Spotify playlist id"
    />
    <br />
    <small>Example: 37i9dQZF1E4s8TIUrRS0Gj</small>
    <br />
    <div :key="index" v-for="(oembedUrl, index) in oembedUrls">
      <iframe
        title="deezer-widget"
        :src="oembedUrl"
        width="100%"
        height="150"
        frameborder="0"
        allowtransparency="true"
        allow="encrypted-media; clipboard-write"
      ></iframe>
    </div>
  </div>
</template>

<script>
import { reactive, toRefs, onMounted, computed, watch } from "vue";
import fetchJsonp from "fetch-jsonp";
import queryString from "query-string";

export default {
  name: "App",
  components: {},

  setup() {
    // TOKENS EXPIRES EVERY 1 HOUR... SO YOU MUST ENTER YOUR SPOTIFY AND DEEZER TOKENS BEFORE START USING THIS!!!
    const SPOTIFY_BASIC_TOKEN =
      "NzZiYjYyZGYxYjM2NGNhMGIwY2QwNTM2YWQ3ZTg2OTY6NDkzZGUzNTM0ZTQxNDNmYmEzZjQ3ZmIyZDJlZTg1NTI=";
    // const SPOTIFY_TOKEN =
    //   "BQDqJujipqKbNEvDqljmtsp6yomC-nUu9mqLGU5SaehHYGW6Q3AldTfu_vUQ1rUK0bOr7gcPnVsnctjzH49X_cl02siERC4S5wsGLFLGQPlEfuK_0k5ZaXJCBKJdSoHyzg-y2dW8DvFZWRC3PCvFS27V1fJstp0";
    const DEEZER_TOKEN = "frRt5PeyToVgdP7jC0cyl75QIqb3EuxGYWOFPDNFvQ7et9U0ut";
    const SPOTIFY_MARKET = "FR";
    const state = reactive({
      id: "",
      deezerTracks: [],
    });

    const oembedUrls = computed(() => {
      return state.deezerTracks
        .filter((track) => track)
        .map((track) => {
          //const oembedUrl = `https://api.deezer.com/oembed?url=https://www.deezer.com/track/${track.id}&maxwidth=700&maxheight=300&tracklist=true&format=json`;
          //const oembedUrl = `https://widget.deezer.com/widget/dark/track/${track.id}?app_id=457142&autoplay=false&radius=true&tracklist=true`;
          const oembedUrl = `https://widget.deezer.com/widget/dark/track/${track.id}?app_id=457142&autoplay=false&radius=false&tracklist=false`;

          return oembedUrl;
        });
    });

    const fetchSpotifyAccessToken = async () => {
      const response = await fetch("https://accounts.spotify.com/api/token", {
        method: "post",
        headers: new Headers({
          Authorization: `Basic ${SPOTIFY_BASIC_TOKEN}`,
          "Content-Type": "application/x-www-form-urlencoded",
        }),
        body: queryString.stringify({
          grant_type: "client_credentials",
        }),
      });
      const result = await response.json();
      console.log(result);
      return result.access_token;
    };

    const fetchSpotifyPlaylist = async () => {
      const spotifyAccessToken = await fetchSpotifyAccessToken();

      const url = `https://api.spotify.com/v1/playlists/${state.id}/tracks?market=${SPOTIFY_MARKET}&limit=100`;
      const response = await fetch(url, {
        method: "get",
        headers: new Headers({
          Authorization: `Bearer ${spotifyAccessToken}`,
          "Content-Type": "application/json",
          Accept: "application/json",
        }),
      });
      const result = await response.json();
      console.log(result, "spotify");
      return result;
    };

    const fetchDeezerSearch = async ({ artist, album, track }) => {
      //const url = `https://cors-anywhere.herokuapp.com/https://api.deezer.com/search?q=artist:"${artist}" album:"${album}" track:"${track}"&access_token=${DEEZER_TOKEN}`;
      const url = `https://api.deezer.com/search?q=artist:"${artist}" album:"${album}" track:"${track}"&access_token=${DEEZER_TOKEN}&output=jsonp`;
      console.log(url);
      const response = await fetchJsonp(url, {
        method: "get",
        headers: new Headers({
          // "Access-Control-Allow-Origin": "*",
          // "Access-Control-Allow-Methods": "GET,PUT,POST,DELETE,PATCH,OPTIONS",
          // Origin: "https://mftnl.csb.app",
          // "X-Requested-With": "XMLHttpRequest",
        }),
      });
      const result = await response.json();
      console.log(result, "deezer");
      return result.data[0];
    };

    const start = async () => {
      if (!state.id.length) {
        return;
      }
      const results = await fetchSpotifyPlaylist(state.id);
      if (results) {
        const tracksInfo = results.items
          .filter((item) => item)
          .map((item) => {
            console.log(item, "item");
            const track = item.track.name;
            const album = item.track.album.name;
            const artist = item.track.artists
              .map((artist) => artist.name)
              .join(", ");
            return { track, album, artist };
          });
        state.deezerTracks = await Promise.all(
          tracksInfo.map(async (trackInfo) => {
            return await fetchDeezerSearch(trackInfo);
          })
        );

        console.log(state.deezerTracks);
      }
    };

    watch(
      () => state.id,
      () => {
        start();
      }
    );

    onMounted(async () => {
      start();
    });

    return {
      ...toRefs(state),
      oembedUrls,
    };
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
