<script>
  import page from "page";
  // eslint-disable-next-line import/no-extraneous-dependencies
  import { fly } from "svelte/transition";

  import { Button } from "@graph-paper/button";
  import { Stack } from "@graph-paper/stack";
  import { Box } from "@graph-paper/box";
  import Logo from "./components/Logo.svelte";
  import OneBigGraph from "./components/OneBigGraph.svelte";
  import Summary from "./components/Summary.svelte";
  import Results from "./components/Results.svelte";
  import Code from "./components/Code.svelte";
  import { site, plotTemplate } from "../public/config";

  let logo;
  let weeklyData, dailyData, studyLength;
  let currentExperiment;
  let currentPage = 0;
  let plots = [];
  let isNav = false;

  let EXPERIMENTER_API_URL = "https://experimenter.services.mozilla.com/api/v1/experiments"

  async function fetchDataByType(normandySlug, type) {
    let source = `statistics_${normandySlug}_${type}.json`;
    let data = await fetch(`data/${source}`).then((r) => r.json()).then((json) => applySplit(json));
    return data;
  }

  async function fetchData(experimentSlug) {
    currentExperiment = await fetchExperimentData(experimentSlug);

    let normandySlug = currentExperiment.normandy_slug.replace(/-/g, '_');;
    dailyData = await fetchDataByType(normandySlug, "daily");
    weeklyData = await fetchDataByType(normandySlug, "weekly");
  }

  async function summary(ctx) {
    currentPage = 0;
    await fetchData(ctx.params.experiment_slug);
  }

  async function results(ctx) {
    currentPage = 1;
    await fetchData(ctx.params.experiment_slug);
  }

  async function fetchExperimentData(slug) {
    let experimentURL = `${EXPERIMENTER_API_URL}/${slug}`;
    let experiment = await fetch(experimentURL).then((r) => r.json());

    let startDate = new Date(experiment.start_date);
    let endDate = new Date(experiment.end_date)
    studyLength = Math.floor(
      (((endDate - startDate) / 1000 / 60 / 60 / 24)
        - experiment.proposed_enrollment) / 7
    );
    return {
        "name": experiment.name,
        "slug": experiment.slug,
        "normandy_slug": experiment.normandy_slug,
        "status": experiment.status,
        "startDate": startDate.toDateString(),
        "endDate": endDate.toDateString(),
        "population": experiment.population,
        "enrollmentPeriod": experiment.proposed_enrollment
      };
  }

  async function applySplit(jsonData) {
    let plotSplitField = plotTemplate["plotSplit"];
    let plotSplitSet = new Set();

    // Create the list of plots (split by statistic)
    // to show up on the left panel.
    jsonData.forEach(dataRow => {
      dataRow["window_index"] = parseInt(dataRow["window_index"]);
      let statistic = dataRow[plotSplitField];
      if (plotSplitSet.has(statistic)) {
        return;
      }

      let plotTemplateCopy = Object.assign({}, plotTemplate);
      plotTemplateCopy["relevantRows"] = statistic;
      plotTemplateCopy["title"] = statistic;
      plotTemplateCopy["subtitle"] = plotTemplateCopy["subtitle"].replace("<statistic>", statistic);
      plotTemplateCopy["key"] = statistic;
      plots.push(plotTemplateCopy);

      plotSplitSet.add(statistic);
    });
    return jsonData;
  }

  page("/:experiment_slug", summary);
  page("/summary/:experiment_slug", summary);
  page("/results/:experiment_slug", results);
  page({ hashbang: true });
</script>

<style>
  h1 {
    padding: 0px;
    margin: 0px;
    font-size: 2em;
    color: var(--cool-gray-800);
  }

  .body {
    padding-bottom: var(--space-8x);
    padding-left: calc(var(--space-1x) * 10);
  }

  .overview_box {
    width: 100%;
    background: #EAEAEA;
    box-shadow: 0px 4px 4px rgba(0, 0, 0, 0.25);
    border-radius: 10px;
  }

  .highlights_box {
    width: 100%;
    padding-top: 25px;
    background: #D8D0D0;
    border-radius: 10px;
  }

  .inline_bold {
    font-weight: bold;
    display:inline;
  }

  .warning {
    width: 100%;
    height: 36px;
    background: rgba(255, 255, 0, 0.62);
    border: 1px solid #000000;
    box-sizing: border-box;
    backdrop-filter: blur(4px);
    /* Note: backdrop-filter has minimal browser support */
    border-radius: 10px;
    text-align: center;
    line-height: 36px;
    font-weight: bold;
    font-style: oblique;
  }

  .grid-container {
    display: grid;
    grid-template-columns: auto auto auto;
    padding-bottom: 10px;
    padding-right: 10px;
    width: 1400px;
  }
  .grid-item {
    padding: 5px;
    padding-left: 30px;
    font-size: 20px;
    text-align: left;
  }
</style>

<svelte:head>
  <title>{site.title}</title>
</svelte:head>
  {#if currentExperiment}
  <nav
    on:mouseover={() => {
      isNav = true;
    }}
    on:mouseleave={() => {
      isNav = false;
    }}>
    <div>
      <Box padding={2}>
        <div style=" opacity: {isNav ? 1 : 0.5}; transition: opacity 300ms;">
          <Logo bind:this={logo} size={80} />
        </div>
      </Box>
      <Box pad={2}>
        <Stack space={1}>
          <Button
            level="low"
            toggled={currentPage === 0}
            href="/summary/{currentExperiment.slug}"
            on:click={() => {
              logo.rebuildLogo();
            }}>
            Summary
          </Button>
          <Button
            level="low"
            toggled={currentPage === 1}
            href="/results/{currentExperiment.slug}"
            on:click={() => {
              logo.rebuildLogo();
            }}>
            Results
          </Button>
        </Stack>
      </Box>
      <!-- footer -->
      {#if site.feedback}
        <Box padding={3}>
          <div
            style="padding-top: var(--space-4x); font-size: var(--text-02); color:
            var(--cool-gray-7500);">
            <div
              style="font-size: var(--text-015); text-transform: uppercase;color:
              var(--cool-gray-550); font-weight: bold;">
              feedback
            </div>
            {site.feedback}
          </div>
        </Box>
      {/if}

    </div>
  </nav>
  {/if}

  <main class="body">
    <Stack space={2}>
      {#if currentExperiment}
      <header>
        <h1>"{currentExperiment.name}" Analysis Report</h1>
      </header>

      <div class="overview_box">
        <div class="subtitle1">
          Overview
        </div>
        <div>
          <div class="grid-container">
            <div class="grid-item">
              <div class="custom_label">Status:</div><span>{currentExperiment.status}</span>
            </div>
            <div class="grid-item">
              <div class="custom_label">Population:</div>{currentExperiment.population}
            </div>
            <div class="grid-item">
              <div class="custom_label">Enrollment Period:</div>{currentExperiment.enrollmentPeriod} days
            </div>
            <div class="grid-item">
              <div class="custom_label">Start Date:</div>{currentExperiment.startDate}
            </div>
            <div class="grid-item">
              <div class="custom_label">End Date:</div>{currentExperiment.endDate}
            </div>
            <div class="grid-item">
              <div class="custom_label">Enrollment Health:</div>???
            </div>
          </div>

          <div class="highlights_box">
            <div class="row">
                <div class="label_large">Hypothesis:</div><div class="text_large">This experiment is expected to <div class="inline_bold">???</div> with <div class="inline_bold">&lt;feature metric&gt;</div> for the treatment group.</div>
            </div>
            <div class="row">
              <div class="row_left">
                <div class="label_large">Result:</div>
                {#if currentExperiment.status === "Live"}
                <b class="text_large" style="color: yellow;">PENDING</b>
                {:else}
                <b class="text_large">???</b>
                {/if}
              </div>
            </div>
          </div>
        </div>
      </div>

      {#if currentExperiment.status === "Live"}
      <div class="warning">
        EXPERIMENT RESULTS ARE IN PROGRESS...
      </div>
      {/if}
      {/if}

      {#if currentPage === 0}
      <Summary weeklyData={weeklyData} studyLength={studyLength}/>
      {:else if currentPage === 1}
      <Results dailyData={dailyData} weeklyData={weeklyData}/>
      {/if}
    </Stack>
  </main>
