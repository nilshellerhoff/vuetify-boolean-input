# Vuetify Boolean Input
A boolean input for [vuetifyjs](https://vuetifyjs.com/) which produces a SQL style condition

![Screenshot from 2021-08-12 23-02-07](https://user-images.githubusercontent.com/24147614/129269427-682e76a4-fca3-45dd-b1ce-81ae24b4899f.png)

This example would select all american or german programmers (given that you group them accordingly).

## Installation
Just copy `BooleanInput.vue` to your components

## Usage

```
<template>
  <div>
    <BooleanInput ref="boolInp" :items="items"></BooleanInput>
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
      items: [
        { text: "Programmer", value: 1 },
        { text: "Male", value: 2 },
        { text: "American", value: 4 },
        { text: "German", value: 5 },
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
- **items (required)**: an array of items, each with `text` and `value` keys:
    ```
    items: [
            { text: "Programmer", value: 1 },
            { text: "Male", value: 2 },
            { text: "American", value: 4 },
            { text: "German", value: 5 },
          ],
    ```
    `text` will be used in the selector, `value` will be used in the expression
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
    - arguments are either `value`s of items, or a subcondition

If using the output of `getSqlExpression()`, remember to sanitize the SQL expression server-side!
