<template>
  <div ref="holder">
    <div class="groupedbarchart">
      <vega-lite :spec="spec" :data="values"></vega-lite>
      <p> {{ chart }} </p>
    </div>
  </div>
</template>


<script>
import { mapState } from 'vuex'
import AugurStats from 'AugurStats'
export default {
  props: ['source', 'citeUrl', 'citeText', 'title', 'disableRollingAverage', 'alwaysByDate', 'data', 'field'],
  data() {
    return {
      values: []
    }
  },
  computed: {
    repo () {
      return this.$store.state.baseRepo
    },
    gitRepo () {
      return this.$store.state.gitRepo
    },
    period () {
      return this.$store.state.trailingAverage
    },
    earliest () {
      return this.$store.state.startDate
    },
    latest () {
      return this.$store.state.endDate
    },
    compare () {
      return this.$store.state.compare
    },
    comparedRepos () {
      return this.$store.state.comparedRepos
    },
    rawWeekly () {
      return this.$store.state.rawWeekly
    },
    showArea () {
      return this.$store.state.showArea
    },
    showTooltip () {
      return this.$store.state.showTooltip
    },
    showDetail () {
      return this.$store.state.showDetail
    },
    months() { return {1: 'Jan', 2: 'Feb', 3: 'Mar', 4: 'Apr', 5: 'May', 6: 'Jun', 7: 'Jul', 8: 'Aug', 9: 'Sep', 10: 'Oct',  11: 'Nov',  12: 'Dec', 
      } 
    },
    spec() {
      let config = {
          "$schema": "https://vega.github.io/schema/vega-lite/v3.json",
          "padding": {'left': 45, 'top': 10, 'right': 70, 'bottom': 50},
          "config": {
              // "view": {"stroke": "transparent"},
              // "axis": {"domainWidth": 1}
              "bar": {
                "discreteBandSize": 10
              }
          },
          "title": {
            "text": this.title,
            // "offset": 55
            "offset": 10
          },
          "width": 420,
          "height": 250,
          "layer": [
            {
              "mark": "text",
              "encoding": {
                  "text": {
                    "field": "month", "type": "nominal",
                    "axis": {"title": ""}
                  },
                  "y": {
                    "field": "net_lines_minus_whitespace", "type": "quantitative",
                    "axis": {"title": "net loc"},
                  },
                  "x": {
                    "field": "commits", "type": 'quantitative',
                    // "field": "name", "type": "ordinal",
                    // "scale": {"rangeStep": 12},
                    "axis": {"title": "commits"},
                    // "sort": {
                    //   "field": "net",
                    //   "op": "mean",
                    //   "order": "descending"
                    // },
                  },
                  "color": {
                    // "value": "red"
                    "field": "month", "type": "ordinal",
                    "scale": {"range": ["black", "#CC0314", "#1403CC","#098118","#CCAE00","#C24F00","#5E0081","#0FA1C1","#BD00B3"]},
                    // "legend": null

                  }
              },
            },
            {

              "mark": "point",
              "encoding": {
                  "y": {
                    "field": "next_y1", "type": "quantitative",
                    "axis": {"title": "", "labels": false},
                  },
                  "x": {
                    "field": "next_x1", "type": 'quantitative',
                    "axis": {"title": "", "labels": false},
                  },
                  "color": {
                    // "value": "red"
                    "field": "month", "type": "ordinal",
                    "scale": {"range": ["black", "#CC0314", "#1403CC","#098118","#CCAE00","#C24F00","#5E0081","#0FA1C1","#BD00B3"]},
                    // "legend": null

                  }
              },
            },
          ],
          "resolve": {"scale": {"y": "independent", "x": "independent"}},
      }
      // Get the repos we need
      let repos = []
      if (this.gitRepo) {
        repos.push(window.AugurAPI.Repo({ gitURL: this.gitRepo }))
      } // end if (this.$store.repo)

            /*
       * Takes a string like "commits,lines_changed:additions+deletions"
       * and makes it into an array of endpoints:
       *
       *   endpoints = ['commits','lines_changed']
       *
       * and a map of the fields wanted from those endpoints:
       *
       *   fields = {
       *     'lines_changed': ['additions', 'deletions']
       *   }
       */
      let endpoints = []
      let fields = {}
      this.source.split(',').forEach((endpointAndFields) => {
        let split = endpointAndFields.split(':')
        endpoints.push(split[0])
        if (split[1]) {
          fields[split[0]] = split[1].split('+')
        }
      })
      if (this.data) {
        processGitData(this.data)
      } else {
        let repo = window.AugurAPI.Repo({ gitURL: this.gitRepo })
        repo[this.source]().then((data) => {
          console.log("batch data", data)
          processData(data)
        }, () => {
          //this.renderError()
        }) // end batch request
      }
      $(this.$el).find('.showme, .hidefirst').removeClass('invis')
      $(this.$el).find('.stackedbarchart').removeClass('loader')
      let processGitData = (data) => {
        let repo = window.AugurAPI.Repo({ gitURL: this.repo })
        let dat = []
        repo.changesByAuthor().then((changes) => {
          dat.push(changes)
        })
      }
      let defaultProcess = (obj, key) => {
            let d = null
            if (typeof(field) == "string") field = [field]
            d = AugurStats.convertKey(obj[key], key)
            return d
          }
      let processData = (data) => {
        console.log(repos, data, "CHECK")
        let sort = this.field == 'commit' ? "patches" : "net"
        data.sort(function(a, b) {
          return b[sort] - a[sort];
        });
        console
        for (let i = 0; i < data.length - 1; i++){
          let diffLoc = data[i+1]['net_lines_minus_whitespace'] - data[i]['net_lines_minus_whitespace']
          let diffCom = data[i+1]['commits'] - data[i]['commits']
          data[i]['next_y1'] = (data[i]['net_lines_minus_whitespace'] + diffLoc / 10)// - (data[i]['net_lines_minus_whitespace'] / 20)
          data[i]['next_x1'] = (data[i]['commits'] + diffCom / 10)// - (data[i]['commits'] / 10)
          data[i]['next_y2'] = (data[i]['net_lines_minus_whitespace'] + diffLoc / 10) //- (data[i]['net_lines_minus_whitespace'] / 10)
          data[i]['next_x2'] = (data[i]['commits'] + diffCom / 10) - (data[i]['commits'] / 20)
          data[i]['next_y3'] = data[i]['net_lines_minus_whitespace'] + ((data[i+1]['net_lines_minus_whitespace'] - data[i]['net_lines_minus_whitespace']) / 10)
          data[i]['next_x3'] = data[i]['commits'] + ((data[i+1]['commits'] - data[i]['commits']) / 10)
        }
        data.forEach((el) => {

          el.month = this.months[el.month]
        })
        repos.forEach((repo) => {
          // let d = defaultProcess(data[repo], Object.keys(data[this.repo])[0])
          // d[0].repo = repo.gitURL ? repo.gitURL : repo.githubURL
          // this.values.push(d[0])
          console.log("repo data", data)
          this.values = data
        })
      }
      return config
    },
  }
}
</script>