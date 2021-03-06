<template>
   <div id="item-editor">
      <div id="editor">
         <div id="item-options">
            <fieldset id="general-options">
               <legend> Item Options </legend>
               <BasicOption id="ID" v-model="items[selectedItem].id"/>
               <BasicOption id="Name" v-model="items[selectedItem].name"/>
               <BasicOption id="Durability" v-model.number="items[selectedItem].durability" type="number"/>
               <BasicOption id="Item Type" v-model="items[selectedItem].type"/>
               <div class="option lore">
                  <div class="label">Lore</div>
                  <textarea rows="2" v-model.lazy="items[selectedItem].lore"></textarea>
               </div>
               <div class="option">
                  <div class="label">Hide Flags</div>
                  <input type="checkbox" v-model="items[selectedItem]['hide-flags']">
               </div>
            </fieldset>

            <fieldset id="attributes">
               <legend> Attributes </legend>
               <div class="attribute" v-for="(display, attribute) in getDisplayNames()">
                     <div class="label">{{ display }}</div>
                     <input class="attribute-input" v-model.number.lazy="items[selectedItem].attributes[attribute]" type="number" step="any">
               </div>
            </fieldset>

            <fieldset id="enchantments">
               <legend> Enchantments </legend>
               <div id="enchantments-list">
                  <div class="enchantment" v-for="enchantment in items[selectedItem].enchantments"> 
                     <div>
                        <div class="label"> Type </div>
                        <input v-model.lazy="enchantment.type"> 
                     </div>
                     <div>
                        <div class="label"> Level </div> 
                        <input class="level" type="number" v-model.number.lazy="enchantment.level">
                     </div>
                  </div>
               </div>
               <div>
                  <button @click="addEnchantment">
                     Add
                  </button>
                  <button @click="removeEnchantment">
                     Remove
                  </button>
               </div>
            </fieldset>
         </div>
         <fieldset id="item-list">
            <legend> Items </legend>
            <ul id="item-list-element">
               <template v-for="(item, index) in items">
                  <li :id="'item-' + index" class="item-element">
                     <div class="item" @click="selectItem(index)"> {{ item.name + " (" + item.id + ")"}} </div>
                     <button class="up-button" @click="swapItem(index, index - 1)"> ^ </button>
                     <button class="down-button" @click="swapItem(index, index + 1)"> v </button>
                     <button class="delete-button" @click="deleteItem(index)"> x </button>
                  </li>
               </template>
            </ul>
            <div id="item-list-controls">
               <button @click="addItem"> Add Item </button>
            </div>
         </fieldset>
      </div>
      <button @click="clearAll">Clear All</button>
      <button @click="downloadJson">Download Items</button>
      <button @click="clickUpload">Upload Items</button>
      <input type="file" @change="readItems" id="upload-config">

      <a id="download"></a>
      <textarea id="copy"></textarea>
   </div>
</template>

<script>
import { downloadFile } from '../util/util';
import BasicOption from '../components/BasicOption';

const DISPLAY_NAMES = {
   "atherys:strength": "Strength",
   "atherys:physical_resistance": "Physical Resistance",
   "atherys:constitution": "Constitution",
   "atherys:magical_resistance": "Magical Resistance",
   "atherys:dexterity": "Dexterity",
   "atherys:base_damage": "Base Damage",
   "atherys:intelligence": "Intelligence",
   "atherys:base_armor": "Base Armor",
   "atherys:wisdom": "Wisdom"
};

export default {
   components: {
      BasicOption
   },

   data () {
      return {
         items: [this.createDefaultItem()],
         selectedItem: 0
      }
   },
   mounted() {
      this.selectItem(0);
   },
   methods: {
      downloadJson() {
         let converted = this.items.map(item => {
            return JSON.parse(JSON.stringify(item, this.replacer));
         })
         downloadFile(JSON.stringify({items: converted}, null, 3), "items.conf", "download");
      },
      readItems(e) {
         const files = e.target.files || e.dataTransfer.files;

         if (!files.length) { 
            return; 
         }
         const reader = new FileReader();
         reader.onload = (e) => {
            this.items = JSON.parse(reader.result).items;
            this.items.forEach(item => {
               if (item.enchantments) {
                  let enchantments = [];

                  Object.keys(item.enchantments).forEach(type => {
                     enchantments.push({ type: type, level: item.enchantments[type] });
                  });

                  item.enchantments = enchantments;
               }

               if (item.lore) {
                  let lore = "";
                  let i = 0;

                  item.lore.forEach(line => {
                     i++;
                     if (i == item.lore.length) {
                        lore += line;
                     } else {
                        lore += line + "\n";
                     }
                  });

                  item.lore = lore;
               }
            });
         }
         reader.readAsText(files[0]);
      },
      clickUpload() {
         document.getElementById("upload-config").click();
      },
      replacer(key, value) {
         if (key === "lore") {
            return value.split("\n");
         }

         if (key === "enchantments") {
            let enchantments = {};
            value.forEach(enchantment => {
               enchantments[enchantment.type] = enchantment.level;
            })
            return enchantments;
         }

         if (value === 0 || value === null || value === undefined) {
            return undefined;
         }

         return value;
      },
      clearAll() {
         Object.assign(this.$data, this.$options.data.call(this));
      },
      copy() {
         let copied = document.getElementById('copy');
         copied.value = this.generated;
         copied.select();
         let bool = document.execCommand("copy");
         console.log(bool);
      },
      addEnchantment() {
         this.items[this.selectedItem].enchantments.push({type:"", level:""});
      },
      removeEnchantment() {
         if (this.items[this.selectedItem].enchantments.length > 0) {
            this.items[this.selectedItem].enchantments.pop();
         }
      },
      addItem() {
         this.items.push(this.createDefaultItem());
      },
      deleteItem(index) {
         if (this.items.length === 1) {
            this.addItem();
         } else if (index <= this.selectedItem && this.selectedItem !== 0) {
            this.selectItem(this.selectedItem - 1);
         }
         this.items.splice(index, 1);
      },
      selectItem(index) {
         document.getElementById("item-" + this.selectedItem).classList.remove("item-selected");
         this.selectedItem = index;
         document.getElementById("item-" + index).classList.add("item-selected");
      },
      swapItem(from, to) {
         if (from < 0 || to < 0 || from > this.items.length || to > this.items.length) return;
         let temp = this.items[from];
         this.items[from] = this.items[to];
         this.items[to] = temp;
         this.$forceUpdate();
      },
      createDefaultItem() {
         return {
            id: "new-item",
            name: "New Item",
            durability: null,
            type: null,
            lore: "",
            attributes: {
               "atherys:constitution": null,
               "atherys:strength": null,
               "atherys:dexterity": null,
               "atherys:intelligence": null,
               "atherys:wisdom": null,
               "atherys:magical_resistance": null,
               "atherys:physical_resistance": null,
               "atherys:base_armor": null,
               "atherys:base_damage": null
            },
            "hide-flags": true,
            enchantments: []
         }
      },
      getDisplayNames() {
         return DISPLAY_NAMES;
      }
   },
   filters: {
      whitespace(value) {
         return value.split(' ').join('');
      },
      capitalize(value) {
         if (!value) return ''
         value = value.toString()
         return value.charAt(0).toUpperCase() + value.slice(1)
      }
   }
}
</script>

<style lang="stylus">
input
   max-height: 30px

button 
   margin-right: 5px

textarea 
   resize: none

#item-editor 
   max-width: 100%
   box-sizing: border-box

#editor 
   display: flex
   margin-bottom: 10px

#item-list
   width: 40%
   display: flex
   flex-direction: column
   justify-content: space-between

#item-list-element ul
   list-style-type: none
   height: 470px
   overflow: auto

.item-element
   display: flex
   justify-content: space-between
   align-items: center
   margin-bottom: 10px
   border-bottom: solid 1px
   padding-bottom: 2px

.item
   cursor: pointer
   flex-grow: 1

.delete-button
   background-color: #E64F4F
   border: none
   color: white
   cursor: pointer
   border-radius: 5px
   height: 20px

.item-selected 
   color: #e2950f

.output-window 
   color: wheat

.value 
   color: cornflowerblue

#download 
   display: none

#copy 
   position: absolute
   left: -9999px

.label 
   display: flex
   align-items: center
   margin-right: 5px
   width: 20%
   min-width: 60px

#enchantments
   display: flex
   flex-direction: column

#enchantments-list
   height: 225px
   overflow: auto

.enchantment
   border-bottom: solid 1px
   margin-bottom: 10px
   padding-bottom: 5px
   > .label
      min-width: 50px
      max-width: 50px
   > input:first-of-type
      margin-right: 10px
   > div
      display: flex
   > div:first-child
      margin-bottom: 5px

.level
   max-width: 50px

.attribute, .option 
   align-items: center
   display: flex

.attribute .label 
   min-width 100px

#item-options 
   display: grid 
   grid-template-columns: 1fr
   grid-gap: 10px
   width: 60%
   margin-right: 10px

#general-options 
   display: grid
   grid-template-columns: repeat(2, 1fr)
   grid-column: 1 / span 2
   grid-gap: 10px
   > div
      margin-bottom: 5px
      > input[type="number"]
         width: 75px
         box-sizing: border-box
      > input
         width: 200px

textarea 
   width: 100%
   height: 300px

#attributes 
   display: grid
   grid-template-columns: repeat(2, 1fr)
   grid-gap: 10px

.attribute-input 
   width: 10%
   min-width: 50px

#upload-config 
   display none

@media (max-width: 1100px)
   #options
      grid-template-columns: 1fr
      grid-template-rows: 1fr 1fr
</style>
