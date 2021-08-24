<template>
  <div>
    <v-row no-gutters>
      <!-- show the operator (every row) -->
      <v-col cols="2">
        <v-select
          :items="logicOperators"
          v-model="condition.logicOperator"
          label="Operator"
          class="pr-2"
          @change="fixChildren(condition)"
        ></v-select>
      </v-col>
      <!-- show subtree if there are two arguments -->
      <template v-if="condition.arguments && condition.arguments.length == 2">
        <v-row
          v-for="argument in condition.arguments"
          :key="argument.id"
          no-gutters
        >
          <v-col offset="1">
            <BooleanInput
              :condition="argument"
              :entities="entities"
            ></BooleanInput>
          </v-col>
        </v-row>
      </template>
      <!-- otherwise show the entity/item selector besides operator -->
      <template v-else>
        <v-col cols="4">
          <v-autocomplete
            :items="entities"
            v-model="condition.entity"
          ></v-autocomplete>
        </v-col>
        <v-col cols="1">
          <v-select
            :items="comparisonOperators"
            v-model="condition.comparisonOperator"
          ></v-select>
        </v-col>
        <v-col cols="4">
          <v-autocomplete
            :items="availableItems"
            v-model="condition.item"
          ></v-autocomplete>
        </v-col>
      </template>
    </v-row>
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
    // entities (required)
    // entities: [
    //   {
    //     text: 'Entity'
    //     value: 'column name'
    //     items: [ // optional if type is number or date
    //       {
    //         text: 'item display name',
    //         value: 'item value / DB id'
    //       }
    //     ],
    //     type: 'number' or 'date' // only if items is not set
    //   }
    // ]
    entities: {
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
          id: new Date().getTime(),
          logicOperator: "",
          comparisonOperator: "=",
          item: this.entities[0].items[0].value,
          entity: this.entities[0].value,
          arguments: [],
        };
      },
    },
  },
  data: () => ({
    // selectable operators
    logicOperators: [
      { value: "", text: "(none)" },
      { value: "AND", text: "AND" },
      { value: "OR", text: "OR" },
    ],
    comparisonOperators: [
      { value: "=", text: "=" },
      { value: "<>", text: "!=" },
    ],
  }),
  computed: {
    // timestamp is used for unique id generation
    time: () => {
      return new Date().getTime();
    },
    // get the available items based on selected entity
    availableItems() {
      // let items = []
      // this.entities.forEach(function(entity) {
      //   if (this.condition.entity == entity.value) {
      //     items = this.entities[0].items
      //   }
      // }, this);
      // return items
      let items = [];
      this.entities.forEach((entity) => {
        items = items.concat(entity.items);
      });
      return items;
    },
  },
  methods: {
    getDefaultCondition(idoffset = 0) {
      return {
        id: new Date().getTime() + idoffset,
        logicOperator: "",
        comparisonOperator: "=",
        item: this.entities[0].items[0].value,
        entity: this.entities[0].value,
        arguments: [],
      };
    },
    addChildren(c) {
      // add two children to the current node (for AND and OR operators)
      c.arguments = [this.getDefaultCondition(), this.getDefaultCondition(1)];
      c.entity = "";
      c.item = "";
    },
    removeChildren(c) {
      // add one child to the current node (for NOT and '' operators)
      c.entity = this.entities[0].value;
      c.item = this.entities[0].items[0].value;
      c.arguments = [];
    },
    fixChildren(c) {
      // check the operator and if the length of arguments is not correct, call the appropriate method
      if (["AND", "OR"].includes(c.logicOperator)) {
        if (c.arguments.length != 2) {
          this.addChildren(c);
        }
      } else {
        if (c.arguments.length != 0) {
          this.removeChildren(c);
        }
      }
    },
    getSqlExpression(column, condition = this.condition) {
      // get an SQL expression for the logical conditon
      // REMEMBER: you need to validate this server-side, as it can easily be manipulated during api calls...
      let sql = "";

      if (condition.arguments.length == 0) {
        // we have a direct entity-value condition
        // add a direct expression (i.e. column=value or column<>value)
        sql += condition.entity + condition.comparisonOperator + condition.item;
      } else {
        // we have a subcondition
        // recursively call this method on each of the subconditions and put parentheses around it
        for (let i = 0; i < condition.arguments.length; i++) {
          sql +=
            "(" + this.getSqlExpression(column, condition.arguments[i]) + ")";

          // if we are in the first operation add the operator (so it stands between the arguments)
          if (i == 0) {
            sql += " " + condition.logicOperator + " ";
          }
        }
      }

      return sql;
    },
  },
  watch: {
    condition: {
      handler() {
        this.$emit("change");
      },
      deep: true,
    },
  },
};
</script>
