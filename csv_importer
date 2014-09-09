<?php
/*
Plugin Name: Csv Data Management
Description: Import the csv file from import.io.
Author: Nidhi
Version: The Plugin's Version 1.0
License: A "Slug" license name e.g. GPL2
*/

class CSVImporterIoPlugin {
    var $defaults = array(
        'headlines'      => null,
        'intro_para'       => null,
        'csv_post_date'       => null,
        'tags'       => null,
        'category' => null,
        'csv_post_author'     => null,
        'csv_post_slug'       => null,
        'csv_post_parent'     => 0,
        'image'=>null,
          '_num'		=>null,
        'csv_post_parent'     => 0,
        '_pageUrl'=>null,
    );

    var $log = array();

    /**
     * Determine value of option $name from database, $default value or $params,
     * save it to the db if needed and return it.
     *
     * @param string $name
     * @param mixed  $default
     * @param array  $params
     * @return string
     */
    function process_option($name, $default, $params) {
        if (array_key_exists($name, $params)) {
            $value = stripslashes($params[$name]);
        } elseif (array_key_exists('_'.$name, $params)) {
            // unchecked checkbox value
            $value = stripslashes($params['_'.$name]);
        } else {
            $value = null;
        }
        $stored_value = get_option($name);
        if ($value == null) {
            if ($stored_value === false) {
                if (is_callable($default) &&
                    method_exists($default[0], $default[1])) {
                    $value = call_user_func($default);
                } else {
                    $value = $default;
                }
                add_option($name, $value);
            } else {
                $value = $stored_value;
            }
        } else {
            if ($stored_value === false) {
                add_option($name, $value);
            } elseif ($stored_value != $value) {
                update_option($name, $value);
            }
        }
        return $value;
    }

    /**
     * Plugin's interface
     *
     * @return void
     */
    function form() {
        $opt_draft = $this->process_option('csv_importer_import_as_draft',
            'publish', $_POST);
        $opt_cat = $this->process_option('csv_importer_cat', 0, $_POST);

        if ('POST' == $_SERVER['REQUEST_METHOD']) {
            $this->post(compact('opt_draft', 'opt_cat'));
        }

        // form HTML {{{
?>

<div class="wrap">
    <h2>Import CSV</h2>
    <form class="add:the-list: validate" method="post" enctype="multipart/form-data">
        <!-- Import as draft -->
         <!-- <p>
        <input name="_csv_importer_import_as_draft" type="hidden" value="publish" />
        <label><input name="csv_importer_import_as_draft" type="checkbox" <?php //if ('draft' == $opt_draft) { echo 'checked="checked"'; } ?> value="draft" /> Import posts as drafts</label>
        </p>-->

        <!-- Parent category
         <p>Organize into category <?php //wp_dropdown_categories(array('show_option_all' => 'Select one ...', 'hide_empty' => 0, 'hierarchical' => 1, 'show_count' => 0, 'name' => 'csv_importer_cat', 'orderby' => 'name', 'selected' => $opt_cat));?><br/>
            <small>This will create new categories inside the category parent you choose.</small></p> -->

        <!-- File input -->
        <p><label for="csv_import">Import CSV Data:</label><br/>
            <input name="csv_import" id="csv_import" type="file" value="" aria-required="true" /></p>
        <p class="submit"><input type="submit" class="button" name="submit" value="Import" /></p>
    </form>
</div><!-- end wrap -->

<?php
        // end form HTML }}}

    }

    function print_messages() {
        if (!empty($this->log)) {

        // messages HTML {{{
?>

<div class="wrap">
    <?php if (!empty($this->log['error'])): ?>

    <div class="error">

        <?php foreach ($this->log['error'] as $error): ?>
            <p><?php echo $error; ?></p>
        <?php endforeach; ?>

    </div>

    <?php endif; ?>

    <?php if (!empty($this->log['notice'])): ?>

    <div class="updated fade">

        <?php foreach ($this->log['notice'] as $notice): ?>
            <p><?php echo $notice; ?></p>
        <?php endforeach; ?>

    </div>

    <?php endif; ?>
</div><!-- end wrap -->

<?php
        // end messages HTML }}}

            $this->log = array();
        }
    }

    /**
     * Handle POST submission
     *
     * @param array $options
     * @return void
     */
    function post($options) {
        if (empty($_FILES['csv_import']['tmp_name'])) {
            $this->log['error'][] = 'No file uploaded, aborting.';
            $this->print_messages();
            return;
        }
        
    if ($_FILES['csv_import']['tmp_name'] != "") {
        $file_tmp = $_FILES['csv_import']['tmp_name'];
        $file_name = time() . $_FILES['csv_import']['name'];

        $ext = substr($file_name, strrpos($file_name, '.') + 1);
        if ($ext != "csv") {
			
			$this->log['error'][] = 'Only CSV are allowed to upload.';
            $this->print_messages();
            return;
           
        }
    }

        if (!current_user_can('publish_pages') || !current_user_can('publish_posts')) {
            $this->log['error'][] = 'You don\'t have the permissions to publish posts and pages. Please contact the blog\'s administrator.';
            $this->print_messages();
            return;
        }

        require_once 'File_Import_IO/DataImportIo.php';

        $time_start = microtime(true);
        $csv = new File_CSV_DataSource;
        $file = $_FILES['csv_import']['tmp_name'];
        $this->stripBOM($file);

        if (!$csv->load($file)) {
            $this->log['error'][] = 'Failed to load file, aborting.';
            $this->print_messages();
            return;
        }

        // pad shorter rows with empty values
        $csv->symmetrize();

        // WordPress sets the correct timezone for date functions somewhere
        // in the bowels of wp_insert_post(). We need strtotime() to return
        // correct time before the call to wp_insert_post().
        $tz = get_option('timezone_string');
        if ($tz && function_exists('date_default_timezone_set')) {
            date_default_timezone_set($tz);
        }

        $skipped = 0;
        $imported = 0;
        $comments = 0;
        foreach ($csv->connect() as $csv_data) {
            if ($post_id = $this->create_post($csv_data, $options)) {
                $imported++;
            } else {
                $skipped++;
            }
        }

        if (file_exists($file)) {
            @unlink($file);
        }

        $exec_time = microtime(true) - $time_start;

        if ($skipped) {
            $this->log['notice'][] = "<b>Skipped {$skipped} posts (CSV is not supporting ).</b>";
        }
        $this->log['notice'][] = sprintf("<b>Imported {$imported} posts in %.2f seconds.</b>", $exec_time);
        $this->print_messages();
    }

    function create_post($data, $options) {
		  global $user_ID, $wpdb;

        $opt_draft = isset($options['opt_draft']) ? $options['opt_draft'] : null;
        $opt_cat = isset($options['opt_cat']) ? $options['opt_cat'] : null;
        
       $data = array_merge($this->defaults, $data);
        /*$type = $data['csv_post_type'] ? $data['csv_post_type'] : 'post';
        $valid_type = (function_exists('post_type_exists') &&
            post_type_exists($type)) || in_array($type, array('post', 'page'));

        if (!$valid_type) {
            $this->log['error']["type-{$type}"] = sprintf(
                'Unknown post type "%s".', $type);
        }*/
        $imagecheck = substr($data['image'], -3);
        $tagname = str_replace('  ', ',', $data['tags']);

   $query =  mysql_query("SELECT distinct wp_posts.ID,wp_postmeta.post_id,wp_postmeta.meta_value,wp_postmeta.meta_key FROM wp_posts left join wp_postmeta on (wp_posts.ID=wp_postmeta.post_id) where wp_postmeta.meta_key ='num_id' and wp_postmeta.meta_value='".trim($data['_num'])."'");
   $resultnum = mysql_num_rows($query);
    if ($resultnum > 0  ) {
		    $ouput = mysql_fetch_array($query);
		    $ouput['ID']; 	
            $new_post = array(
			'ID' =>  $ouput['ID'],
			'post_title'   => convert_chars($data['headlines']),
            'post_content' => wpautop(convert_chars($data['intro_para'])),
            'post_status'  => 'publish',
            'post_type'      => 'post',
            'post_date'    => $this->parse_date($data['csv_post_date']),
            'post_name'    => $data['csv_post_slug'],
            'post_author'  => $this->get_auth_id($data['csv_post_author']),
            );
        if ('page' !== $type) {
           // $new_post['tags_input'] = $data['tags'];
            // Setup categories before inserting - this should make insertion
            // faster, but I don't exactly remember why :) Most likely because
            // we don't assign default cat to post when csv_post_categories
            // is not empty.
            $cats = $this->create_or_get_categories($data, $opt_cat);
            $new_post['post_category'] = $cats['post'];
        }
        
          $id = wp_update_post($new_post);
          
            if($imagecheck=='jpg' || $imagecheck=='png' ) {
            add_post_meta($id, 'image_url',trim($data['image']), true);
            }
            add_post_meta($id, 'num_id',$data['_num'], true); 
            add_post_meta($id, 'page_url',$data['_pageUrl'], true);
            wp_set_post_tags($id, $tagname, true );
      
          
    } else {
     
        $new_post = array(
            'post_title'   => convert_chars($data['headlines']),
            'post_content' => wpautop(convert_chars($data['intro_para'])),
            'post_status'  => 'publish',
            'post_type'      => 'post',
            'post_date'    => $this->parse_date($data['csv_post_date']),
            'post_name'    => $data['csv_post_slug'],
            'post_author'  => $this->get_auth_id($data['csv_post_author']),
        );
        
          // pages don't have tags or categories
        if ('page' !== $type) {
           // $new_post['tags_input'] = $data['tags'];
           
            // Setup categories before inserting - this should make insertion
            // faster, but I don't exactly remember why :) Most likely because
            // we don't assign default cat to post when csv_post_categories
            // is not empty.
            $cats = $this->create_or_get_categories($data, $opt_cat);
            $new_post['post_category'] = $cats['post'];
        }

        
          $id = wp_insert_post($new_post);
          if($imagecheck=='jpg' || $imagecheck=='png' ) {
            add_post_meta($id, 'image_url',trim($data['image']), true);
            }
            add_post_meta($id, 'num_id',$data['_num'], true); 
            add_post_meta($id, 'page_url',$data['_pageUrl'], true);
            wp_set_post_tags($id, $tagname, true );
      
  
}

    
        if ('page' !== $type && !$id) {
            // cleanup new categories on failure
            foreach ($cats['cleanup'] as $c) {
                wp_delete_term($c, 'category');
            }
        }
        return $id;
    }

    /**
     * Return an array of category ids for a post.
     *
     * @param string  $data csv_post_categories cell contents
     * @param integer $common_parent_id common parent id for all categories
     * @return array category ids
     */
    function create_or_get_categories($data, $common_parent_id) {
        $ids = array(
            'post' => array(),
            'cleanup' => array(),
        );
        $items = array_map('trim', explode(',', $data['category']));
        foreach ($items as $item) {
            if (is_numeric($item)) {
                if (get_category($item) !== null) {
                    $ids['post'][] = $item;
                } else {
                    $this->log['error'][] = "Category ID {$item} does not exist, skipping.";
                }
            } else {
                $parent_id = $common_parent_id;
                // item can be a single category name or a string such as
                // Parent > Child > Grandchild
                $categories = array_map('trim', explode('>', $item));
                if (count($categories) > 1 && is_numeric($categories[0])) {
                    $parent_id = $categories[0];
                    if (get_category($parent_id) !== null) {
                        // valid id, everything's ok
                        $categories = array_slice($categories, 1);
                    } else {
                        $this->log['error'][] = "Category ID {$parent_id} does not exist, skipping.";
                        continue;
                    }
                }
                foreach ($categories as $category) {
                    if ($category) {
                        $term = $this->term_exists($category, 'category', $parent_id);
                        if ($term) {
                            $term_id = $term['term_id'];
                        } else {
                            $term_id = wp_insert_category(array(
                                'cat_name' => $category,
                                'category_parent' => 142,
                            ));
                            $ids['cleanup'][] = $term_id;
                        }
                        $parent_id = $term_id;
                    }
                }
                $ids['post'][] = $term_id;
            }
        }
        return $ids;
    }

    /**
     * Parse taxonomy data from the file
     *
     * array(
     *      // hierarchical taxonomy name => ID array
     *      'my taxonomy 1' => array(1, 2, 3, ...),
     *      // non-hierarchical taxonomy name => term names string
     *      'my taxonomy 2' => array('term1', 'term2', ...),
     * )
     *
     * @param array $data
     * @return array
     */
    function get_taxonomies($data) {
        $taxonomies = array();
        foreach ($data as $k => $v) {
            if (preg_match('/^csv_ctax_(.*)$/', $k, $matches)) {
                $t_name = $matches[1];
                if ($this->taxonomy_exists($t_name)) {
                    $taxonomies[$t_name] = $this->create_terms($t_name,
                        $data[$k]);
                } else {
                    $this->log['error'][] = "Unknown taxonomy $t_name";
                }
            }
        }
        return $taxonomies;
    }

    /**
     * Return an array of term IDs for hierarchical taxonomies or the original
     * string from CSV for non-hierarchical taxonomies. The original string
     * should have the same format as csv_post_tags.
     *
     * @param string $taxonomy
     * @param string $field
     * @return mixed
     */
    function create_terms($taxonomy, $field) {
        if (is_taxonomy_hierarchical($taxonomy)) {
            $term_ids = array();
            foreach ($this->_parse_tax($field) as $row) {
                list($parent, $child) = $row;
                $parent_ok = true;
                if ($parent) {
                    $parent_info = $this->term_exists($parent, $taxonomy);
                    if (!$parent_info) {
                        // create parent
                        $parent_info = wp_insert_term($parent, $taxonomy);
                    }
                    if (!is_wp_error($parent_info)) {
                        $parent_id = $parent_info['term_id'];
                    } else {
                        // could not find or create parent
                        $parent_ok = false;
                    }
                } else {
                    $parent_id = 0;
                }

                if ($parent_ok) {
                    $child_info = $this->term_exists($child, $taxonomy, $parent_id);
                    if (!$child_info) {
                        // create child
                        $child_info = wp_insert_term($child, $taxonomy,
                            array('parent' => $parent_id));
                    }
                    if (!is_wp_error($child_info)) {
                        $term_ids[] = $child_info['term_id'];
                    }
                }
            }
            return $term_ids;
        } else {
            return $field;
        }
    }

    /**
     * Compatibility wrapper for WordPress term lookup.
     */
    function term_exists($term, $taxonomy = '', $parent = 0) {
        if (function_exists('term_exists')) { // 3.0 or later
            return term_exists($term, $taxonomy, $parent);
        } else {
            return is_term($term, $taxonomy, $parent);
        }
    }

    /**
     * Compatibility wrapper for WordPress taxonomy lookup.
     */
    function taxonomy_exists($taxonomy) {
        if (function_exists('taxonomy_exists')) { // 3.0 or later
            return taxonomy_exists($taxonomy);
        } else {
            return is_taxonomy($taxonomy);
        }
    }

    /**
     * Hierarchical taxonomy fields are tiny CSV files in their own right.
     *
     * @param string $field
     * @return array
     */
    function _parse_tax($field) {
        $data = array();
        if (function_exists('str_getcsv')) { // PHP 5 >= 5.3.0
            $lines = $this->split_lines($field);

            foreach ($lines as $line) {
                $data[] = str_getcsv($line, ',', '"');
            }
        } else {
            // Use temp files for older PHP versions. Reusing the tmp file for
            // the duration of the script might be faster, but not necessarily
            // significant.
            $handle = tmpfile();
            fwrite($handle, $field);
            fseek($handle, 0);

            while (($r = fgetcsv($handle, 999999, ',', '"')) !== false) {
                $data[] = $r;
            }
            fclose($handle);
        }
        return $data;
    }

    /**
     * Try to split lines of text correctly regardless of the platform the text
     * is coming from.
     */
    function split_lines($text) {
        $lines = preg_split("/(\r\n|\n|\r)/", $text);
        return $lines;
    }

    

    function get_auth_id($author) {
        if (is_numeric($author)) {
            return $author;
        }

        // get_userdatabylogin is deprecated as of 3.3.0
        if (function_exists('get_user_by')) {
            $author_data = get_user_by('login', $author);
        } else {
            $author_data = get_userdatabylogin($author);
        }

        return ($author_data) ? $author_data->ID : 0;
    }

    /**
     * Convert date in CSV file to 1999-12-31 23:52:00 format
     *
     * @param string $data
     * @return string
     */
    function parse_date($data) {
        $timestamp = strtotime($data);
        if (false === $timestamp) {
            return '';
        } else {
            return date('Y-m-d H:i:s', $timestamp);
        }
    }

    /**
     * Delete BOM from UTF-8 file.
     *
     * @param string $fname
     * @return void
     */
    function stripBOM($fname) {
        $res = fopen($fname, 'rb');
        if (false !== $res) {
            $bytes = fread($res, 3);
            if ($bytes == pack('CCC', 0xef, 0xbb, 0xbf)) {
                $this->log['notice'][] = 'Getting rid of byte order mark...';
                fclose($res);

                $contents = file_get_contents($fname);
                if (false === $contents) {
                    trigger_error('Failed to get file contents.', E_USER_WARNING);
                }
                $contents = substr($contents, 3);
                $success = file_put_contents($fname, $contents);
                if (false === $success) {
                    trigger_error('Failed to put file contents.', E_USER_WARNING);
                }
            } else {
                fclose($res);
            }	
        } else {
            $this->log['error'][] = 'Failed to open file, aborting.';
        }
    }
}


function csv_import_io() {
    require_once ABSPATH . '/wp-admin/admin.php';
    $plugin = new CSVImporterIoPlugin;
    add_submenu_page('options-general.php', 'CSV Importer', 'Csv Data Management','manage_options', __FILE__,array($plugin, 'form'));
}

add_action('admin_menu', 'csv_import_io');

?>
