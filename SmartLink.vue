<template>
  <component
      v-on="$listeners"
      v-bind="bindings"
      :is="current_component">
    <slot/>
  </component>
</template>

<script>
/* 

NOTE: if you're using SSR, don't use v-html on this component. 
instead, put a <span> in the <slot> and add the v-html to that. 
Otherwise, hydration will bail when it tries to appendChild into a comment /shrug

See methods.checkLinkIsInertia method to define which href's ARE inertia-enabled

Usage:

<smart-link :href="/my-inertia-route"> => <inertia-link :href="/my-inertia-route" />

<smart-link :href="/my-non-inertia-route"> => <a :href="/my-non-inertia-route" />

<smart-link :hide_empty="true" :href="null"> => <fragment />

<smart-link :fallback_tag="div" :href="null">This Texts is Sometimes Linked</smart-link> => <div>This Texts is Sometimes Linked</div>

<smart-link :href="/some-page">
Link Text Here
</smart-link>

*/

import {Link} from '@inertiajs/inertia-vue'

// 'https://mysite.com'; // OPTIONAL define the base url for url building. otherwise, it'll use window.location.hostname
const BASE_URL = null; 
// OPTIONAL define common inertia-link :only="" param for all inertia-links here...
const COMMON_INERTIA_ONLY = null; 

export default {
  components: {'inertia-link':Link},
  props: {
    // defaults to _self, can be _blank, etc
    target: {required: false, type: String, default: null },

    href: {required: false, type: String, default: null},

    // what tag to use if href is falsey
    fallback_tag: {default: 'span'},

    // whether to return an empty fragement if falsey href is provided
    hide_empty: {type: Boolean, default: false}
  },
  data() {
    // debugging values
    return {
      has_listener: null,
      targets_new: null,
      is_mailto: null,
      contains_hash: null,
      is_absolute: null,
      is_external: null,
      final_target: null,
      final_href: null,
      is_inertia_route: null,
    }
  },
  computed: {
    current_component(){
      let {current_component} = this.analyzeHref();
      return current_component;
    },
    // bindings as object to prevent prenting nulls to DOM
    bindings(){
      let _bindings = {}
      if(this.final_target && this.final_target.trim().length){
        _bindings['target'] = this.final_target;
      }
      if(this.final_href && this.final_href.trim().length){
        _bindings['href'] = this.final_href;
      }
      return _bindings;
    }
  },
  methods:{
    checkUrlIsInertia(href){
      if(!href || !href.length){
        return false
      }
      // define what routes are "inertia-enabled"
      if(href.indexOf('/my-inertia-route')>-1){
        return true;
      }
      return false;
    },
    // returns 'a'(anchor), 'inertia-link', 'span', 'fragment'
    analyzeHref(){
      let href = this?.href ?? '';
      // mailto:// use <a>
      const is_mailto = href.indexOf('mailto:') > -1;
      // if it has a hash, use <a>
      const contains_hash = href.indexOf('#') > -1;
      const is_absolute = false; // TODO: sniff href to see if it's relative or absolute (does it contain :// protocol already?)
      const is_external = false; // TODO: sniff url and check to see if it's internal or external to the current TLD
      
      // a fallback tag for when you assign :href prop to a variable that can be empty/null/falsey
      let fallback_tag = this.fallback_tag ?? 'span' // default to wrapping it in a span
      
      let current_component = 'inertia-link';
      let bind_attrs = {...this.$listeners};

      // peek into $listeners and check to see if a v-on:x is set
      // if so, keep it a regular anchor
      let has_listener = false;
      // console.log('listeners?',this.$listeners);
      // loop over this.$listeners.entries
      for (const [key, value] of Object.entries(this.$listeners)) {
        //console.log(`smartlink $listeners ${key}: ${value}`);
        if(value){
          has_listener = true;
        }
      }

      if(href?.length){
        // console.log('href?',href);
        // TODO: need to make it valid first by prepending base_url if relative
        let temp_href = href;
        if(temp_href.indexOf('://') === -1){
          temp_href = `${BASE_URL ?? window?.location?.protocol + '//' + window?.location?.host}${temp_href}`
        }
        let links_to_current_page = false;
        let the_hash = null;
        try{
          const href_as_URL = new URL(temp_href);
          links_to_current_page = href_as_URL.pathname === window?.location?.pathname;
          // console.log('same page?',href_as_URL.pathname,window.location.pathname,links_to_current_page)
          the_hash = href_as_URL.hash
        }catch(e){
          console.error(e);
        }


        // if the link contains a hash and is the SAME page we're already viewing,
        // change the href to a v-on:click.prevent and call scroll to hash element or use document.getElementByID(theHash).scrollIntoView({behavior:'smooth'})
        if(contains_hash && links_to_current_page){
          let el = document.getElementById(`#${the_hash}`)
            if(el){
              bind_attrs['click'] = (e)=>{
                e.preventDefault();
                el.scrollIntoView({behavior:'smooth'})
              }
            }else{
              console.warn('el not found for hash',the_hash)
            }
        }
      }

      let target = this.target ?? '_self';
      if(href?.length){
        bind_attrs['target'] = this?.target ?? '_self';
      }

      const final_target = bind_attrs['target']

      let targets_new = final_target?.indexOf('new') > -1 ||
                        final_target?.indexOf('blank') > -1;

      // we are phasing-in inertial on a per-route basis, so we need to limit them here for now
      const is_inertia_route = this.checkUrlIsInertia(href)

      if (
        has_listener
        || (
          href.length
          && (
            targets_new
            || is_mailto
            || contains_hash
            || !is_inertia_route 
          )
        )
      ) {
        // default to a regular anchor in these cases
        current_component = 'a'
      } else {
        current_component = href.length
            ? 'inertia-link'
            : (
                this.props?.hide_empty
                    ? 'fragment'
                    : fallback_tag
            )
      }

      if(current_component === 'inertia-link'){
        // OPTIONAL if you have global "only" param to apply to all your inertia links, apply them here now
        if(COMMON_INERTIA_ONLY){
          bind_attrs['only'] = COMMON_INERTIA_ONLY
        }
      }

      // expose results of analysis for debugging via vue devtools / js-console
      // could also console log them, but it becomes hard to read,
      // easier to target a specific link, then see its results
      this.final_href = href;
      this.has_listener = has_listener;
      this.targets_new = targets_new;
      this.is_mailto = is_mailto;
      this.contains_hash = contains_hash;
      this.is_absolute = is_absolute;
      this.is_external = is_external;
      this.final_target = final_target;
      this.is_inertia_route = is_inertia_route;

      return {
        current_component
      }
    }
  }
}
</script>
