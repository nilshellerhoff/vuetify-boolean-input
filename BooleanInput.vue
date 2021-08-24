<template>
  <div>
    <v-row no-gutters style="position: relative">
      <!-- show the operator (every row) -->
      <v-col cols="2">
        <v-select
          :items="logicOperators"
          v-model="condition.logicOperator"
          label="Operator"
          class="pr-2"
          dense
          @change="fixChildren(condition)"
        ></v-select>
      </v-col>
      <!-- show subtree if there are two arguments -->
      <template v-if="condition.arguments && condition.arguments.length == 2">
        <!-- for each argument recursively call BooleanInput -->
        <v-row
          v-for="argument in condition.arguments"
          :key="argument.id"
          no-gutters
          style="position: relative"
        >
          <div
            style="
              position: absolute;
              left: 8px;
              top: 18px;
              width: calc(5%);
              height: 2px;
              background-color: #888;
            "
          ></div>
          <div
            style="
              position: absolute;
              left: 8px;
              top: 10px;
              width: 2px;
              height: 10px;
              background-color: #888;
            "
          ></div>
          <v-col offset="1">
            <BooleanInput
              :condition="argument"
              :entities="entities"
            ></BooleanInput>
          </v-col>
        </v-row>
      </template>
      <!-- otherwise show the entity/item selector besides operator -->
      <!-- entity selector -->
      <template v-else>
        <v-col cols="4">
          <v-autocomplete
            :items="entities"
            v-model="condition.entity"
            dense
            class="pr-2"
          ></v-autocomplete>
        </v-col>
        <!-- comparison operator selector -->
        <v-col cols="1">
          <v-select
            :items="comparisonOperators[currEntity.type]"
            v-model="condition.comparisonOperator"
            dense
            class="pr-2"
          ></v-select>
        </v-col>
        <!-- value input -->
        <v-col cols="4">
          <!-- item selector -->
          <v-autocomplete
            v-if="currEntity.type == 'item'"
            :items="currEntity.items"
            v-model="condition.value"
            dense
            class="pr-2"
          ></v-autocomplete>
          <!-- number input -->
          <v-text-field
            v-if="currEntity.type == 'number'"
            v-model="condition.value"
            type="number"
            dense
            class="pr-2"
          ></v-text-field>
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
    //     type: 'item' or 'number'
    //     items: [ // if type is 'item'
    //       {
    //         text: 'item display name',
    //         value: 'item value / DB id'
    //       }
    //     ],
    //   }
    // ]
    entities: {
      type: Array,
      required: true,
    },

    // condition parameter is optional, default is items[0] is true
    //
    // condition {
    //   id: some number,
    //
    //   -- if a single comparison
    //   operator: "=" or "<>"
    //   entity: value of the selected entity
    //   value: selected value (item, number ...)

    //   -- else if a subcondition
    //   logicOperator: "", "AND", "OR"
    //   arguments: [
    //     at least two subconditions
    //   ]
    // }
    condition: {
      type: Object,
      default: function () {
        return {
          id: new Date().getTime(),
          comparisonOperator: "=",
          value: this.entities[0].items[0].value,
          entity: this.entities[0].value,
          logicOperator: "none",
          arguments: [],
        };
      },
    },
  },
  data: () => ({
    // selectable operators
    logicOperators: [
      { value: "none", text: "(none)" },
      { value: "AND", text: "AND" },
      { value: "OR", text: "OR" },
    ],
    comparisonOperators: {
      item: [
        { text: "=", value: "=" },
        { text: "≠", value: "<>" },
      ],
      number: [
        { text: "=", value: "=" },
        { text: "≠", value: "<>" },
        { text: "≥", value: ">=" },
        { text: "≤", value: "<=" },
        { text: ">", value: ">" },
        { text: "<", value: "<" },
      ],
    },
  }),
  computed: {
    // timestamp is used for unique id generation
    time: () => {
      return new Date().getTime();
    },
    // get all the info of the currently selected entity
    currEntity() {
      let currEntity = null;
      this.entities.forEach((entity) => {
        if (entity.value == this.condition.entity) {
          currEntity = entity;
        }
      });
      return currEntity;
    },
  },
  methods: {
    getDefaultCondition(idoffset = 0) {
      return {
        id: new Date().getTime() + idoffset,
        logicOperator: "",
        comparisonOperator: "=",
        entity: this.entities[0].value,
        value: this.entities[0].items[0].value,
        arguments: [],
      };
    },
    addChildren(c) {
      // add two children to the current node (for AND and OR operators)
      c.arguments = [this.getDefaultCondition(), this.getDefaultCondition(1)];
      c.entity = "";
      c.value = "";
    },
    removeChildren(c) {
      // add one child to the current node (for NOT and '' operators)
      c.entity = this.entities[0].value;
      c.value = this.entities[0].items[0].value;
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
        sql += condition.entity + condition.comparisonOperator + condition.value;
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
