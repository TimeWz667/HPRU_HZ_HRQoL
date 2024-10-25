<template>
  <div class="loss">
    <p>
      Value set: {{ pars.ValueSet }}
    </p>
    <input type="range" name="age0" value="70" v-model="age" min="30" max="95" step="1"><p>Age: {{ age }}</p>
    <p>Life exp: {{ proj.le.toFixed(1) }}</p>
    <p>Quality-adjusted life exp: {{ proj.qale.toFixed(1) }}</p>
    <p>Quality-adjusted life exp with HZ: {{ mlu(proj.qale_hz, 1) }}</p>
    <p>QALY loss: {{ mlu(proj.loss, 3) }}</p>
    <p>QALY loss from perfect health: {{ mlu(proj.loss_perfect, 3) }}</p>
  </div>

</template>

<script>
  const pars = require("../public/pars_qol_uk.json");
  import {quantileSeq} from "mathjs";

  export default {
    data() {
      return {
        pars: pars,
        age: 70,
        time_pf: 0.035,
        proj: {
          le: 0,
          qale: 0,
          qale_hz: new Array(100).fill(0),
          loss: new Array(100).fill(0),
          loss_perfect: new Array(100).fill(0)
        }
      }
    },
    mounted() {
      this.update(this.age, this.time_pf);
    },
    watch: {
      age() {
        this.update(this.age, this.time_pf);
      }
    },
    methods: {
      update(age0, dis) {
        this.proj.le = this.calc_le(age0).le;
        this.proj.qale = this.calc_qale(age0, dis).le;
        this.proj.qale_hz = this.calc_qale_hz(age0, dis).le;
        this.proj.loss = this.proj.qale_hz.map(d => this.proj.qale - d);
        this.proj.loss_perfect = this.calc_ql_perfect_hz(age0, dis).le;
      },
      calc_le(age0) {
        return pars.pars_demo.filter(d => d.Age >= age0)
          .reduce((a, curr) => {
            a.le += a.surv * (1 - curr.r_death / 2);
            a.surv = a.surv * (1 - curr.r_death);
            return a;
          }, {le: 0, surv: 1});
      },
      calc_qale(age0, dis) {
        return pars.pars_demo.filter(d => d.Age >= age0)
          .reduce((a, curr) => {
            a.le += a.surv * (1 - curr.r_death / 2) * curr.norm * a.delta;
            a.surv *= (1 - curr.r_death);
            a.delta /= (1 + dis);
            return a;
          }, {le: 0, surv: 1, delta: 1});
      },
      calc_qale_hz(age0, dis) {
        return pars.pars_demo.filter(d => d.Age >= age0)
          .reduce((a, curr) => {
            pars.pars_shortfall.forEach((d, i) => {
              const q = Math.min(curr.norm, d.Q_0) * d.Prop_0 + Math.min(curr.norm, d.Q_1) * d.Prop_1 + Math.min(curr.norm, d.Q_2) * d.Prop_2;
              const rate = d.r0 * Math.exp(age0 * d.ba1);
              const p_health = 1 + (Math.exp(-rate * (curr.Age - age0 + 1)) - Math.exp(-rate * (curr.Age - age0))) / rate;

              a.le[i] += a.surv * (1 - curr.r_death / 2) * (curr.norm * p_health + q * (1 - p_health)) * a.delta;
            })
            a.surv *= (1 - curr.r_death);
            a.delta /= (1 + dis);
            return a;
          }, {le: new Array(pars.pars_shortfall.length).fill(0), surv: 1, delta: 1});
      },
      calc_ql_perfect_hz(age0, dis) {
        return pars.pars_demo.filter(d => d.Age >= age0)
          .reduce((a, curr) => {
            pars.pars_shortfall.forEach((d, i) => {
              const q = d.Q_0 * d.Prop_0 + d.Q_1 * d.Prop_1 + d.Q_2 * d.Prop_2;
              const rate = d.r0 * Math.exp(age0 * d.ba1);
              const p_health = 1 + (Math.exp(-rate * (curr.Age - age0 + 1)) - Math.exp(-rate * (curr.Age - age0))) / rate;
              a.le[i] += a.surv * (1 - curr.r_death / 2) * (1 - q) * (1 - p_health) * a.delta;
            });

            a.surv *= (1 - curr.r_death);
            a.delta /= (1 + dis);
            return a;
          }, {le: new Array(pars.pars_shortfall.length).fill(0), surv: 1, delta: 1});
      },
      mlu(xs, digit) {
        digit |= 1;
        const stats = {
          m: quantileSeq(xs, 0.5),
          l: quantileSeq(xs, 0.025),
          u: quantileSeq(xs, 0.975)
        };
        return `${stats.m.toFixed(digit)} (95% CrI ${stats.l.toFixed(digit)}, ${stats.u.toFixed(digit)})`
      }
    }
  }
</script>
