<template>
  <input ref="input" class="typeahead-suggestions"
         :class="classes"
         :id = "id"
         v-bind:value="value"
         v-on:input="updateValue($event.target.value)"
         v-on:blur="formatValue"
         :placeholder="placeholder"
          >
</template>
<script>
  require('imports-loader?define=>false!typeahead.js/dist/typeahead.jquery.js');
  var Bloodhound = require('imports-loader?define=>false!typeahead.js/dist/bloodhound.js');
  export default {
    data: function() {
      var id =  'typeahead-suggestion' + parseInt(Math.random() *100000);
      return {
        id,
        defaultSuggestions: [],
        query: ''
      };
    },

    props: {
      value: {
        type: String,
        default: ''
      },
      classes: {
        type: String,
        default: 'input'
      },
      displayKey: {
        type: String,
        default: ''
      },
      suggestionTemplate: {
        type: String,
        default: ''
      },
      name: {
        type: String,
        default: 'Vue Auto Complete'
      },
      prefetch: {
        type: String,
        default: ''
      },
      defaultSuggestion: {
        type: Boolean,
        default: false
      },
      remote: {
        type: String,
        default: ''
      },

      remoteAttrs : {
        type: Object,
        default : () => {},
      },
      placeholder: {
        type: String,
        default: ''
      },
      local: {
        type: Array,
        default: function () {
          return []
        }
      },
      responseWrapper: {
        type: String,
        default: ''
      }
    },
    watch:{

      remoteAttrs() {
        this.resetTypeahead()
      },

      local: function(newVal) {
        if(this.defaultSuggestion) {

          this.defaultSuggestions = [...newVal];
        }
        this.resetTypeahead();
//        console.log('watching');
      }
    },

    mounted: function() {
      this.initTypeahead();
      if(this.local.length) {
        this.defaultSuggestions = [...this.local];
      }
    },

    methods: {

      buildRemoteUrl(url) {
        if (typeof this.remoteAttrs !== 'undefined' && Object.keys(this.remoteAttrs).length > 0) {
          var args = []
          for (let attr in this.remoteAttrs) {
            args.push(attr + '=' + this.remoteAttrs[attr])
          }

          url += '&' + args.join('&')
        }
        return url
      },

      updateValue: function (value) {
        this.$emit('input', value)
      },

      formatValue: function () {
        this.$refs.input.value = this.value;
      },

      transformer: function (response) {
        if(this.responseWrapper) {
          response = response[this.responseWrapper];
        }
        if(this.defaultSuggestion && this.local.length === 0) {
          this.defaultSuggestions = response.splice(0, 5);
        }

        return response;
      },

      bloodhoundOption: function(){
        var bloodhoundConfig = {};
        if(this.prefetch) {
          var prefetch = {
            cache: false,
            url: this.prefetch
          };
          if(this.defaultSuggestion) {
            prefetch = {...prefetch, transform: this.transformer};
          }
          bloodhoundConfig = { prefetch};
        }
        if(this.local) {
          bloodhoundConfig = {
            local: this.local,
            ...bloodhoundConfig
          }
        }
        if(this.remote) {
          bloodhoundConfig = {
            remote: {
              url: this.buildRemoteUrl(this.remote),
              wildcard: '%QUERY',
              transform: this.transformer
            },
            ...bloodhoundConfig
          }
        }
        return bloodhoundConfig;
      },

      parseTemplate: function(data) {
        var res = Vue.compile(this.suggestionTemplate);
        var vm =new Vue({
          data,
          render: res.render,
          staticRenderFns: res.staticRenderFns
        }).$mount();

        return vm.$el;
      },

      getSource: function(){
        var self = this;
        var bloodhoundConfig = this.bloodhoundOption();
        var datumTokenizer = this.displayKey ? Bloodhound.tokenizers.obj.whitespace(this.displayKey)
                        :  Bloodhound.tokenizers.whitespace;

        var engine = new Bloodhound({
          datumTokenizer,
          queryTokenizer: Bloodhound.tokenizers.whitespace,
          ...bloodhoundConfig
        });

        var source = function(q, sync, async) {
          if (q === '' && self.defaultSuggestions.length>0 && self.defaultSuggestion) {
//            console.log('default')
            sync(self.defaultSuggestions);
          } else {
            engine.search(q, sync, async);
          }

        };

        return this.defaultSuggestion ? source : engine;
      },

      resetTypeahead: function() {
        $(document).find('#' + this.id).typeahead('destroy');
        this.initTypeahead();
      },

      initTypeahead: function() {
        var self =  this;
        var templates = {};
        if(this.suggestionTemplate) {
          templates = {suggestion: self.parseTemplate}
        };
        var dataset = {
          name: 'Typeahead-Suggestion',
          display: this.displayKey,
          source: this.getSource(),
          limit: Infinity,
          templates
        };
        $(document).find('#' + self.id).typeahead({
            minLength: 0,
            highlight: true
          }, dataset)
        .on('typeahead:select', function(event, suggession) {
          self.$emit('input', self.displayKey? suggession[self.displayKey]: suggession)
          self.$emit('selected', suggession);
        });
      }
    }
  }
</script>