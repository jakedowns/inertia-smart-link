<template>
  <component
      v-on="$listeners"
      v-bind="bindings"
      :is="current_component">
    <slot/>
  </component>
</template>

<script>
/* NOTE: if you're using SSR, don't use v-html on this component. instead, put a <span> in the <slot> and add the v-html to that. Otherwise, hydration will bail when it tries to appendChild onto a comment */

/* define a class that checks if the requested href is an Inertia-enabled page, or not, so the component knows when to render a <a> vs a <inertia-link> */
import { checkUrlIsInertia } from 'checkUrlIsInertia'

import {Link} from '@inertiajs/inertia-vue'

const BASE_URL = null; // 'https://mysite.com'; // OPTIONAL define the base url for url building. otherwise, it'll use window.location.hostname
const COMMON_INERTIA_ONLY = null; // OPTIONAL define common inertia-link :only="" param for all inertia-links here...

export default {
  components: {'inertia-link':Link},
  props: {
    target: {required: false, type: String, default: null },
    href: {required: false, type: String, default: null},
    fallback_tag: {default: 'span'},
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
      const is_inertia_route = checkUrlIsInertia(href)

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

      // d.json({
      //   // t: this,
      //   p: this.props,
      //   href,
      //   has_listener,
      //   targets_new,
      //   is_mailto,
      //   contains_hash,
      //   is_absolute,
      //   is_external,
      //   is_cross_house,
      //   is_same_site_wrong_env,
      //   current_component,
      //   final_target,
      //   text: this.$slots?.default?.[0]?.elm?.innerHTML
      // })

      // expose results of analysis for debugging via vue devtools / js-console
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
