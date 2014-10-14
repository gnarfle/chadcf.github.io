---
layout: post
title: Creating custom filters with drupal and views
created: 1297210874
---
Ah views. Yes, the views module for Drupal is incredibly powerful and combined with CCK has essentially eliminated the need to ever create custom modules for content. While this is a happy thing, sometimes views just doesn't quite do what you want. And, rather sadly, there is almost no documentation on how to create custom views plugins/styles/filters/etc. Boo. 

For a recent project, I had the need to have a view filtered by a single dropdown menu that could contain either taxonomy terms or date parameters. While drupal supports exposed filters for both of these things, it does not support having them both in one select menu. My initial attempt was to just create a block that through some voodoo found and submitted the view with arguments for the view. This almost worked, but tended to screw up ajax pagination, and was rather fragile to begin with. So, I embarked on the path of determining how one might go about creating a truly custom filter for a view. 

<h4>Step 1 - Create your module</h4>

<b><i>custom_filter.module</i></b>
<code>
<?php
function custom_filter_views_api() {
  return array(
    'api' => 2,
    'path' => drupal_get_path('module', 'custom_filter') . '/inc'
  );
}
?>
</code>

And that's about it... All this really says is my module targets views version 2, and you can find the files in the 'inc' directory of my module folder. 

<h4>Step 2 - Module Definitions </h2>

Next we need to tell views about what our module provides.

<b><i>inc/custom_filter.views.inc</i></b>
<code>

<?php
/**
 * Implementation of hook_views_handlers() to register all of the basic handlers
 * views uses.
 */
function custom_filter_views_handlers() {
  return array(
    'info' => array(
      'path' => drupal_get_path('module', 'custom_filter') . '/inc', // path to view files
    ),
    'handlers' => array(
      // register our custom filter, with the class/file name and parent class
      'custom_filters_filter_multiple' => array(
        'parent' => 'views_handler_filter',
      )
    ),
  );
}
</code>

The above definition essentially tells views what our module does. In this case we simply give it the path to our files, and register one handler named 'custom_filters_filter_multiple. The parent parameter tells views which class our handler extends. In this case, I just extended the root filter class.

Next we need to provide a views_data() hook, which tells views a bit of information about our filter.

<b><i>inc/custom_filter.views.inc</i></b>
<code>
function custom_filter_views_data() {
  $data = array();
  
  // The flexible date filter.
  $data['node']['custom_filter'] = array(
    'group' => t('Custom'),
    'real field'  => 'custom_filter',
    'title' => t('Custom Date/Term combined filter'),
    'help' => t('Filter any Views based on date and term'),
    'filter' => array(
      'handler' => 'custom_filters_filter_multiple'
    ),
  ); 
  
  return $data;
}
</code>

In this case we return an array with our filter data. We give it a group (the title of which appears in the drop down to filter, uh, filters when you add a new filter to your view). We give it a real field, which in this instance does not actually exist since we're not filtering on a single real field. The title and help are displayed in the views UI, and the handler again points to our class for implementing this filter. Views seems to like redundency...

<h4>Step 3 - Filter Definition </h4>

Now it's time to actually create our filter. We create a file which contains our class, both matching the names we provided above. 

<i><b>inc/custom_filters_filter_multiple.inc</b></i>
<code>

<?php

class custom_filters_filter_multiple extends views_handler_filter {
  
  /* this method is used to create the options form for the Views UI when creating a view
   * we use the standard drupal form api to return a form array, with the settings
   * we want to capture. 
   */
  function options_form(&$form, &$form_state) {
    parent::options_form($form, $form_state);

    // Step 1: fetch all our vocabularies, and build an array of options
    $terms = taxonomy_get_vocabularies();
    $show = array();
    foreach($terms as $term) {
      $show[$term->vid] = $term->name;
    }
    
    // Step 2: create a select field to choose Vocabulary options
    //   this allows you to choose which vocabulary to fetch terms for in the exposed filter
    $form['filter_vocab'] = array(
      '#type' => 'select',
      '#title'  => t('Vocabulary'),
      '#options'  => $show,
      '#default_value'  => $this->options['filter_vocab']
    );
    
    // Step 3: Create a checkbox field to select whether date options should be included
    $form['include_dates'] = array(
      '#type' => 'checkbox',
      '#title'  => t('Include Date Filters'),
      '#default_value'  => $this->options['include_dates']
    );
  }
  
  /* I'll be perfectly honest, I have no idea if this is required or not. I *think* it may be
   * as it defines our filter field. However I don't use it. I added it when trying to get
   * things working...
   */
  function value_form(&$form, &$form_state) {
    $form['custom_filter']  = array(
      '#type' => 'textfield'
    );
  }
  
  /* A custom display for our exposed form. Views normally uses the value_form for this
   * however we're skipping that entirely since we want our exposed form to be a completely
   * different beast. It's entirely possible I could move this to value_form however...
   */
  function exposed_form(&$form, &$form_state) {
    // for my use case the filtering is controlled by javascript, which
    // submits the form and also handles a date select popup.
    drupal_add_js(drupal_get_path('module','custom_filter') . '/custom_filter.js');
    
    // if we're displaying date options, add them to the options for our output filter
    if ( $this->options['include_dates'] ) {
      $display = array(
        'all'               => t('Show all'),
        'last-four-weeks'   => t('Last four weeks'),
        'last-three-months' => t('Last three months'),
        'last-six-months'   => t('Last six months'),
        'last-year'         => t('Last year'),
        'advanced'          => t('Advanced'),
      );    
    } else {
      $display = array('all'  => t('Show all'));
    }
    
    // get the terms for our configured vocabulary and add them to the options
    // for my case, terms are only 1 level deep, will need changes
    // if you have nested terms.
    if ( $this->options['filter_vocab'] ) {
      $terms = taxonomy_get_tree($this->options['filter_vocab']);
      
      foreach( $terms as $term ) {
        $display[$term->tid] = $term->name;
      }
    }
    
    // now create our select element with the chosen options
    $form['custom_filter'] = array(
      '#type' => 'select',
      '#title'  => t('Browse By'),
      '#options'  => $display
    );
  }

  // the query method is responsible for actually running our exposed filter
  function query() {
    // make sure our base table is included in the query.
    // base table for this is node so it may be redundent...
    $this->ensure_my_table();

    // make sure term node is joined in if needed
    // not exactly optimal since we may not need it if we're filtering by date
    $this->query->add_table('term_node');
    
    // get the value of the submitted filter
    $value = $this->value[0];

    // a bit ugly. Since we have date and taxonomy options we need to do a switch to exhaust
    // the date options before we can assume it's a taxonomy term
    switch( $value ) {
      case 'all';
        return;
      case 'last-four-weeks':
        $this->query->add_where($this->options['group'], "node.created > %s", strtotime('4 weeks ago'));
        break;
      case 'last-three-months':
        $this->query->add_where($this->options['group'], "node.created > %s", strtotime('3 months ago'));
        break;
      case 'last-six-months':
        $this->query->add_where($this->options['group'], "node.created > %s", strtotime('6 months ago'));
        break;
      case 'last-year':
        $this->query->add_where($this->options['group'], "node.created > %s", strtotime('1 year ago'));
        break;
      default:
        if ( is_numeric($value) ) {
          $this->query->add_where($this->options['group'], "term_node.tid = %d", $value);
        }
    }
  }
}
</code>

Hopefully that all makes sense. I'm not entirely sure it's the proper way to do it, but it works well and does what I need!
