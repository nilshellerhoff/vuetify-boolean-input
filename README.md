# Vuetify Boolean Input
A boolean logic input for [vuetifyjs](https://vuetifyjs.com/) which produces a SQL style condition. Should easily be portable to other frameworks like Quasar... if you rework the template part.

![image](https://user-images.githubusercontent.com/24147614/133210013-b9a58e04-c16b-483f-9796-c80ab0134b2b.png)

This example would select all programmers, accountants and pilots who are 30 years or older.

## Installation
Just copy `BooleanInput.vue` to your components

## Usage

```
<template>
  <div>
    <BooleanInput ref="boolInp" :entities="entities"></BooleanInput>
    <v-row>
      <v-col cols="2">
        <v-btn @click="getSql">get SQL</v-btn>
      </v-col>
      <v-col cols="10">
        <v-text-field disabled v-model="sql"></v-text-field>
      </v-col>
    </v-row>
  </div>
</template>

<script>
import BooleanInput from "@/components/BooleanInput.vue";

export default {
  name: "Name",
  components: {
    BooleanInput,
  },
  data: function () {
    return {
      entities: [
        {
          text: "Profession",
          value: "s_profession",
          items: [
            { text: "Programmer", value: 1 },
            { text: "Accounter", value: 2 },
          ],
        },
        {
          text: "Nationality",
          value: "s_nationality",
          items: [
            { text: "American", value: 4, entity: "s_nationality" },
            { text: "German", value: 5, entity: "s_nationality" },
          ],
        },
      ],
      sql: "",
    };
  },
  methods: {
    getSql() {
      return (this.sql = this.$refs.boolInp.getSqlExpression("i_group"));
    },
  },
};
</script>
```
This will show the input form with a text field below which (on click) contains the evaluated SQL expression

### Props of BooleanInput
- **entities (required)**: an array of entities, each with `text` and `value` and `items` keys:
    ```
    entities: [
            { text: "Profession", value: "s_profession", items: (items) },
            { text: "Nationality", value: "s_nationality", items: (items) },
          ],
    ```
    `text` will be used in the selector, `value` will be used in the expression. `items` is an array containing the items selectable for this entity, again with `text` and `value` keys. E.g. `items` for 'Nationality':
    ```
    items: [
      { text: "American", value: "US" },
      { text: "German", value: "DE" },
      { text: "French", value: "FR" },
      ...
    ]
    ```
    Again, `text` will be the value shown in the UI and `value` will be used in the expression. `value` can be the values of a column or e.g. an ID if the expression is used in a join.
- **condition**: an object which represents a binary expression tree
    
    the condition from the screenshot above would be represented as follows:
    ```
    {
      "id": 1,
      "operator": "AND",
      "arguments": [
        {
          "id": 11,
          "operator": "",
          "arguments": [1]
        },
        {
          "id": 12,
          "operator": "OR",
          "arguments": [
            {
              "id": 121,
              "operator": "",
              "arguments": [4]
            },
            {
              "id": 122,
              "operator": "",
              "arguments": [5]
            }
          ]
        }
      ]
    }
    ```
    - IDs can be anything, just need to be unique (if created from the UI timestamps are used).
    - Operators are "" (for =), "NOT" (for <>), "AND" and "OR"
    - arguments are either values of items, or a subcondition

If using the output of `getSqlExpression()`, remember to sanitize the SQL expression server-side!

## TODOs
- [ ] ability to parse SQL-expression into binary tree
- [ ] AND & OR operators with more than two arguments
- [x] add graphical tree representation
- [x] entity-value pairs and other operators (>, < ...)
