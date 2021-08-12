<template>
  <div>
    <v-row no-gutters>
      <!-- show the operator (every row) -->
      <v-col cols="2">
        <v-select
          :items="operators"
          v-model="condition.operator"
          label="Operator"
          class="pr-2"
          @change="fixChildren(condition)"
        ></v-select>
      </v-col>
      <!-- show the item selector besides operator (only if operator requires only one input) -->
      <v-col cols="4">
        <v-autocomplete
          v-if="condition.arguments.length == 1"
          :items="items"
          v-model="condition.arguments[0]"
        ></v-autocomplete>
      </v-col>
    </v-row>
    <!-- otherwise show subtree -->
    <template v-if="condition.arguments.length == 2">
      <v-row
        v-for="argument in condition.arguments"
        :key="argument.id"
        no-gutters
      >
        <v-col offset="1">
          <BooleanInput :condition="argument" :items="items"></BooleanInput>
        </v-col>
      </v-row>
    </template>
  </div>
</template>

<script>
import BooleanInput from "./BooleanInput.vue";

export default {
  name: "BooleanInput",
  compontents: {
    BooleanInput,
  },
  props: {
    // items parameter is required
    //
    // items: [
    //   {
    //     text: 'EntityName',
    //     value: 'DB id ... (what you want in your expression)'
    //   }
    // ]
    items: {
      type: Array,
      required: true,
    },

    // condition parameter is optional, default is items[0] is true
    //
    // condition {
    //   id: 1,
    //   operator: "AND", "OR", "NOT" or ""
    //   arguments: [
    //     a value from items.value    OR
    //     some sub condition
    //   ]
    // }
    condition: {
      type: Object,
      default: function () {
        return {
          id: 1,
          operator: "",
          arguments: [this.items[0].value],
        };
      },
    },
  },
  data: () => ({
    operators: [
      // selectable operators
      { value: "", text: "(none)" },
      { value: "NOT", text: "NOT" },
      { value: "AND", text: "AND" },
      { value: "OR", text: "OR" },
    ],
  }),
  computed: {
    // timestamp is used for unique id generation
    time: () => {
      return new Date().getTime();
    },
  },
  methods: {
    addChildren(c) {
      // add two children to the current node (for AND and OR operators)
      c.arguments = [
        {
          id: this.time,
          operator: "",
          arguments: [this.items[0].value],
        },
        {
          id: this.time + 1,
          operator: "",
          arguments: [this.items[0].value],
        },
      ];
    },
    removeChildren(c) {
      // add one child to the current node (for NOT and '' operators)
      c.arguments = [this.items[0].value];
    },
    fixChildren(c) {
      // check the operator and if the length of arguments is not correct, call the appropriate method
      if (["AND", "OR"].includes(c.operator)) {
        if (c.arguments.length != 2) {
          this.addChildren(c);
        }
      } else {
        if (c.arguments.length != 1) {
          this.removeChildren(c);
        }
      }
    },
    getSqlExpression(column, condition = this.condition) {
      // get an SQL expression for the logical conditon
      // REMEMBER: you need to validate this server-side, as it can easily be manipulated during api calls...
      let sql = "";
      
      for (let i = 0; i < condition.arguments.length; i++) {
        let arg = condition.arguments[i];

        if (typeof arg == "object") {
          // arg is object, which means that we have a subcondition
          // recursively call this method on the subcondition and put parentheses around it
          sql += "(" + this.getSqlExpression(column, arg) + ")";

          // if we are in the first operation add the operator (so it stands between the arguments)
          if (i == 0) {
            sql += " " + condition.operator + " ";
          }
        } else {
          // otherwise we have a value 
          // add a direct expression (i.e. column=value or column<>value)
          let sign = condition.operator == "NOT" ? "<>" : "=";
          sql += column + sign + arg;
        }
      }

      return sql;
    },
  },
};
</script>
