<template>
  <div class="loss">
    <p>
      Value set: {{ pars.ValueSet }}
    </p>
    <input type="number" name="age0" value="70" v-model="age" min="30" max="95" step="1">
    <p>Life exp: {{calc_le(age).le}}</p>
    <p>Quality-adjusted life exp: {{calc_qale(age, time_pf).le}}</p>
  </div>

</template>

<script>
  const pars = require("../public/pars_qol_uk.json");

  export default {
    data() {
      return {
        pars: pars,
        age: 70,
        time_pf: 0.035
      }
    },
    methods: {
      calc_ql(age0, dis) {
        let demo = pars.pars_demo.filter((d) => d.Age >= age0);


      },
      calc_le(age0) {
        return pars.pars_demo.filter(d => d.Age >= age0)
          .reduce((a, curr) => {
            a.surv = a.surv * (1 - curr.r_death);
            a.le += a.surv;
            return a;
          }, {le: 0, surv: 1});
      },
      calc_qale(age0, dis) {
        return pars.pars_demo.filter(d => d.Age >= age0)
          .reduce((a, curr) => {
            a.surv *= (1 - curr.r_death);
            a.delta /= (1 + dis);
            a.le += a.surv * curr.norm * a.delta;
            return a;
          }, {le: 0, surv: 1, delta: 1 + dis});
      }
    }
  }
</script>
