LDAP data source with full CRUD support
=======================================

by analogrithems on November 08, 2009

When I first started, I realized that CakePHP didn't have an LDAP data
source officially supported yet. I did find two articles about some
good attempts. One by `euphrate`_, unfortunately this one was only for
reading from ldap. The second one was by `Gservat`_, this one was a
bit more complete, but was not really working for me.
Before we get started I want to state the environment I was using to
do my work was Redhat Enterprise 5.2 & Fedora 10 (Work requirement)
with redhat directory server 8.1 and Fedora directory server 1.2. Now
while LDAP is a standard protocol, some of the driver may have become
centric to those platforms, so if this is the case, please leave me a
comment and I will try to correct the ldap data source I'm working on.
My hope is to get ldap as an offical CakePHP data source. Some of the
next features I want to implement is data associations. Basically has
and belongs to many relations. This way when you look up an user
account it also shows you all the groups that user is in.


First things first, here is my ldap data source for CakePHP. Place the
following in 'app/models/datasources/ldap_source.php' P.S. You can
always find the most current version in my github as I maintains this
for an opensource LDAP tool I'l working on @ `http://github.com/analog
rithems/idbroker/blob/master/models/datasources/ldap_source.php`_

Model Class:
````````````

::

    <?php 
    /**
     * LdapSource
     * @author euphrate_ylb (base class + "R" in CRUD)
     * @author gservat (aka znoG) ("C", "U", "D" in CRUD)
     * @date 07/2007 (updated 04/2008)
     * @license GPL
     */
    class LdapSource extends DataSource {
        var $description = "Ldap Data Source";
        var $cacheSources = true;
        var $SchemaResults = false;
        var $database = false;
        var $count = 0;
        var $model;
    
        //for formal querys
        var $_result = false;
        
        var $_baseConfig = array (
            'host' => 'localhost',
            'port' => 389,
            'version' => 3
        );
        
        var $_multiMasterUse = 0;
        var $__descriptions = array();
        
        // Lifecycle --------------------------------------------------------------
        /**
         * Constructor
         */
        function __construct($config = null) {
            $this->debug = Configure :: read() > 0;
            $this->fullDebug = Configure :: read() > 1;
            parent::__construct($config);
            return $this->connect();
        }
        
        /**
         * Destructor. Closes connection to the database.
         *
         */
        function __destruct() {
            $this->close();
            parent :: __destruct();
        }
        
        // I know this looks funny, and for other data sources this is necessary but for LDAP, we just return the name of the field we're passed as an argument
        function name( $field ) {
            return $field;
        }
        
        // Connection --------------------------------------------------------------
        function connect($bindDN = null, $passwd = null) {
            $config = $this->config;
            $this->connected = false;
            $hasFailover = false;
    	if(isset($config['host']) && is_array($config['host']) ){
    		$config['host'] = $config['host'][$this->_multiMasterUse];
    		if(count($this->config['host']) > (1 + $this->_multiMasterUse) ) {
    			$hasFailOver = true;
    		}
    	}
    	$bindDN     =  (empty($bindDN)) ? $config['login'] : $bindDN;
    	$bindPasswd =  (empty($passwd)) ? $config['password'] : $passwd;
    	$this->database = @ldap_connect($config['host']);
    	if(!$this->database){
    	    //Try Next Server Listed
    	    if($hasFailover){
    		$this->log('Trying Next LDAP Server in list:'.$this->config['host'][$this->_multiMasterUse],'ldap.error');
    		$this->_multiMasterUse++;
    		$this->connect($bindDN, $passwd);
    		if($this->connected){
    			return $this->connected;
    		}
    	    }
    	}
    
    	//Set our protocol version usually version 3
    	ldap_set_option($this->database, LDAP_OPT_PROTOCOL_VERSION, $config['version']);		
    	// From Filipee, to allow the user to specify in the db config to use TLS
    	// 'tls'=> true in config/database.php
    	if ($config['tls']) {
    		if (!ldap_start_tls($this->database)) {
    			$this->log("Ldap_start_tls failed", 'ldap.error');
    			fatal_error("Ldap_start_tls failed");
    		}
    	}
    	//So little known fact, if your php-ldap lib is built against openldap like pretty much every linux
    	//distro out their like redhat, suse etc. The connect doesn't acutally happen when you call ldap_connect
    	//it happens when you call ldap_bind.  So if you are using failover then you have to test here also.
    	$bind_result = @ldap_bind($this->database, $bindDN, $bindPasswd);
            if (!$bind_result){
    		if(ldap_errno($this->database) == 49){
    			$this->log("Auth failed for '$bindDN'!",'ldap.error');
    		}else{
    			$this->log('Trying Next LDAP Server in list:'.$this->config['host'][$this->_multiMasterUse],'ldap.error');
    			$this->_multiMasterUse++;
    			$this->connect($bindDN, $passwd);
    			if($this->connected){
    				return $this->connected;
    			}
    		}
    
    	}else{
    		 $this->connected = true;
    	}
            return $this->connected;
        }
    
        /**
         * Test if the dn/passwd combo is valid
         */
        function auth( $dn, $passwd ){
    	$this->connect($dn, $passwd);
            if ($this->connected){
    	    return true;
    	}else{
    	    $this->log("Auth Error: for '$dn': ".$this->lastError(),'ldap.error');
    	    return $this->lastError();
    	}
        }
    
        
        /**
         * Disconnects database, kills the connection and says the connection is closed,
         * and if DEBUG is turned on, the log for this object is shown.
         *
         */
        function close() {
            if ($this->fullDebug && Configure :: read() > 1) {
                $this->showLog();
            }
            $this->disconnect();
        }
        
        function disconnect() {
            @ldap_free_result($this->results);
            $this->connected = !@ldap_unbind($this->database);
            return !$this->connected;
        }
        
        /**
         * Checks if it's connected to the database
         *
         * @return boolean True if the database is connected, else false
         */
        function isConnected() {
            return $this->connected;
        }
        
        /**
         * Reconnects to database server with optional new settings
         *
         * @param array $config An array defining the new configuration settings
         * @return boolean True on success, false on failure
         */
        function reconnect($config = null) {
            $this->disconnect();
            if ($config != null) {
                $this->config = am($this->_baseConfig, $this->config, $config);
            }
            return $this->connect();
        }
    
        // CRUD --------------------------------------------------------------
        /**
         * The "C" in CRUD
         *
         * @param Model $model
         * @param array $fields containing the field names
         * @param array $values containing the fields' values
         * @return true on success, false on error
         */
        function create( &$model, $fields = null, $values = null ) {
    		$basedn = $this->config['basedn'];
    		$key = $model->primaryKey;
    		$table = $model->useTable;
            $fieldsData = array();
            $id = null;
            $objectclasses = null;
    
            if ($fields == null) {
                unset($fields, $values);
                $fields = array_keys($model->data);
                $values = array_values($model->data);
            }
            
            $count = count($fields);
            
            for ($i = 0; $i < $count; $i++) {
                if ($fields[$i] == $key) {
                    $id = $values[$i];
                }elseif($fields[$i] == 'cn'){
    				$cn = $values[$i];
    	    }
    	    $fieldsData[$fields[$i]] = $values[$i];
            }
    
    		//Lets make our DN, this is made from the useTable & basedn + primary key. Logically this corelate to LDAP
    	
    		if(isset($table) && preg_match('/=/', $table)){
    			$table = $table.', ';
    		}else{ $table = ''; }
    		if(isset($key) && !empty($key)){
    			$key = "$key=$id, ";
    		}else{ 
    			//Almost everything has a cn, this is a good fall back.
    			$key = "cn=$cn, "; 
    		}
    		$dn = $key.$table.$basedn;
    		
    		$res = @ ldap_add( $this->database, $dn, $fieldsData );
            // Add the entry
            if( $res ){
    	    $model->setInsertID($id);
    	    $model->id = $id;
                return true;
            } else {
    	    $this->log("Failed to add ldap entry: dn:$dn\nData:".print_r($fieldsData,true)."\n".ldap_error($this->database),'ldap.error');
                $model->onError();
                return false;
            }
        }
    	
    	/**
    	 * Returns the query
    	 *
    	 */
    	function query($find, $query = null, $model){
    		if(isset($query[0]) && is_array($query[0])){
    			$query = $query[0];
    		}
    		
    		if(isset($find)){
    		    switch($find){
    			case 'auth':
    				return $this->auth($query['dn'], $query['password']);
    			case 'findSchema':
    				$query = $this->__getLDAPschema();
    				//$this->findSchema($query);
    				break;
    			case 'findConfig':
    				return $this->config;
    				break;
    			default:
    				$query = $this->read($model, $query);
    				break;
    			}
    		}
    		return $query;
    	}
        /**
         * The "R" in CRUD
         *
         * @param Model $model
         * @param array $queryData
         * @param integer $recursive Number of levels of association
         * @return unknown
         */
        function read( &$model, $queryData = array(), $recursive = null ) {
    	$this->model = $model;
            $this->__scrubQueryData($queryData);
            if (!is_null($recursive)) {
                $_recursive = $model->recursive;
                $model->recursive = $recursive;
            }
    
            // Check if we are doing a 'count' .. this is kinda ugly but i couldn't find a better way to do this, yet
            if ( is_string( $queryData['fields'] ) && $queryData['fields'] == 'COUNT(*) AS ' . $this->name( 'count' ) ) {
                $queryData['fields'] = array();
            }
    
            // Prepare query data ------------------------ 
            $queryData['conditions'] = $this->_conditions( $queryData['conditions'], $model);
            if(empty($queryData['targetDn'])){
            	$queryData['targetDn'] = $model->useTable;
            }
            $queryData['type'] = 'search';
            
            if (empty($queryData['order']))
                    $queryData['order'] = array($model->primaryKey);
                        
            // Associations links --------------------------
            foreach ($model->__associations as $type) {
                foreach ($model->{$type} as $assoc => $assocData) {
                    if ($model->recursive > -1) {
                        $linkModel = & $model->{$assoc};
                        $linkedModels[] = $type . '/' . $assoc;
                    }
                }
            }
        
            // Execute search query ------------------------
            $res = $this->_executeQuery($queryData );
            
            if ($this->lastNumRows()==0) 
                return false;
            
            // Format results  -----------------------------
            ldap_sort($this->database, $res, $queryData['order'][0]);
            $resultSet = ldap_get_entries($this->database, $res);
            $resultSet = $this->_ldapFormat($model, $resultSet);    
        	
            // Query on linked models  ----------------------
            if ($model->recursive > 0) {
                foreach ($model->__associations as $type) {
        
                    foreach ($model->{$type} as $assoc => $assocData) {
                        $db = null;
                        $linkModel = & $model->{$assoc};
        
                        if ($model->useDbConfig == $linkModel->useDbConfig) {
                            $db = & $this;
                        } else {
                            $db = & ConnectionManager :: getDataSource($linkModel->useDbConfig);
                        }
        
                        if (isset ($db) && $db != null) {
                            $stack = array ($assoc);
                            $array = array ();
                            $db->queryAssociation($model, $linkModel, $type, $assoc, $assocData, $array, true, $resultSet, $model->recursive - 1, $stack);
                            unset ($db);
                        }
                    }
                }
            }
            
            if (!is_null($recursive)) {
                $model->recursive = $_recursive;
            }
    
            // Add the count field to the resultSet (needed by find() to work out how many entries we got back .. used when $model->exists() is called)
            $resultSet[0][0]['count'] = $this->lastNumRows();
            return $resultSet;
        }
    
        /**
         * The "U" in CRUD
         */
        function update( &$model, $fields = null, $values = null ) {
            $fieldsData = array();
    
            if ($fields == null) {
                unset($fields, $values);
                $fields = array_keys( $model->data );
                $values = array_values( $model->data );
            }
            
            for ($i = 0; $i < count( $fields ); $i++) {
                $fieldsData[$fields[$i]] = $values[$i];
            }
            
    		//set our scope
            $queryData['scope'] = 'base';
    	if($model->primaryKey == 'dn'){
    		$queryData['targetDn'] = $model->id;
    	}elseif(isset($model->useTable) && !empty($model->useTable)){
    		$queryData['targetDn'] = $model->primaryKey.'='.$model->id.', '.$model->useTable;
    	}
        
            // fetch the record
            // Find the user we will update as we need their dn
            $resultSet = $this->read( $model, $queryData, $model->recursive );
            
    	//now we need to find out what's different about the old entry and the new one and only changes those parts
    	$current = $resultSet[0][$model->alias];
    	$update = $model->data[$model->alias];
    
    	foreach( $update as $attr => $value){
    		if(isset($update[$attr]) && !empty($update[$attr])){
    			$entry[$attr] = $update[$attr];
    		}elseif(!empty($current[$attr]) && (isset($update[$attr]) && empty($update[$attr])) ){
    			$entry[$attr] = array();
    		}
    	}
    
    	//if this isn't a password reset, then remove the password field to avoid constraint violations...
    	if(!$this->in_arrayi('userpassword', $update)){
    		unset($entry['userpassword']);
    	}
    	unset($entry['count']);
    	unset($entry['dn']);
    
            if( $resultSet) {
                $_dn = $resultSet[0][$model->alias]['dn'];
                
                if( @ldap_modify( $this->database, $_dn, $entry ) ) {
                    return true;
                }else{
    		$this->log("Error updating $_dn: ".ldap_error($this->database)."\nHere is what I sent: ".print_r($entry,true), 'ldap.error');
    		return false;
    	    }
            }
            
            // If we get this far, something went horribly wrong ..
            $model->onError();
            return false;
        }
    
        /**
         * The "D" in CRUD
         */    
        function delete( &$model ) {
            // Boolean to determine if we want to recursively delete or not
            //$recursive = true;
            $recursive = false;
        
    	if(preg_match('/dn/i', $model->primaryKey)){
    		$dn = $model->id;
    	}else{
    		// Find the user we will update as we need their dn
    		if( $model->defaultObjectClass ) {
    		    $options['conditions'] = sprintf( '(&(objectclass=%s)(%s=%s))', $model->defaultObjectClass, $model->primaryKey, $model->id );
    		} else {
    		    $options['conditions'] = sprintf( '%s=%s', $model->primaryKey, $model->id );
    		}
    		$options['targetDn'] = $model->useTable;
    		$options['scope'] = 'sub';
    
    		$entry = $this->read( $model, $options, $model->recursive );
    		$dn = $entry[0][$model->name]['dn'];
    	}
    
            if( $dn ) {
                if( $recursive === true ) {
                    // Recursively delete LDAP entries
                    if( $this->__deleteRecursively( $dn ) ) {
                        return true;
                    }
                } else {
                    // Single entry delete
                    if( @ldap_delete( $this->database, $dn ) ) {
                        return true;
                    }
                }
            }
            
            $model->onError();
    	$errMsg = ldap_error($this->database);
    	$this->log("Failed Trying to delete: $dn \nLdap Erro:$errMsg",'ldap.error');
            return false;
        }
        
        /* Courtesy of gabriel at hrz dot uni-marburg dot de @ http://ar.php.net/ldap_delete */
        function __deleteRecursively( $_dn ) {
            // Search for sub entries
            $subentries = ldap_list( $this->database, $_dn, "objectClass=*", array() );
            $info = ldap_get_entries( $this->database, $subentries );
            for( $i = 0; $i < $info['count']; $i++ ) {
                // deleting recursively sub entries
                $result = $this->__deleteRecursively( $info[$i]['dn'] );
                if( !$result ) {
                    return false;
                }
            }
            
            return( @ldap_delete( $this->database, $_dn ) );
        }
            
        // Public --------------------------------------------------------------    
        function generateAssociationQuery(& $model, & $linkModel, $type, $association = null, $assocData = array (), & $queryData, $external = false, & $resultSet) {
            $this->__scrubQueryData($queryData);
            
            switch ($type) {
                case 'hasOne' :
                    $id = $resultSet[$model->name][$model->primaryKey];
                    $queryData['conditions'] = trim($assocData['foreignKey']) . '=' . trim($id);
                    $queryData['targetDn'] = $linkModel->useTable;
                    $queryData['type'] = 'search';
                    $queryData['limit'] = 1;
                    return $queryData;
                    
                case 'belongsTo' :
                    $id = $resultSet[$model->name][$assocData['foreignKey']];
                    $queryData['conditions'] = trim($linkModel->primaryKey).'='.trim($id);
                    $queryData['targetDn'] = $linkModel->useTable;
                    $queryData['type'] = 'search';
                    $queryData['limit'] = 1;
    
                    return $queryData;
                    
                case 'hasMany' :
                    $id = $resultSet[$model->name][$model->primaryKey];
                    $queryData['conditions'] = trim($assocData['foreignKey']) . '=' . trim($id);
                    $queryData['targetDn'] = $linkModel->useTable;
                    $queryData['type'] = 'search';
                    $queryData['limit'] = $assocData['limit'];
    
                    return $queryData;
    
                case 'hasAndBelongsToMany' :
                    return null;
            }
            return null;
        }
    
        function queryAssociation(& $model, & $linkModel, $type, $association, $assocData, & $queryData, $external = false, & $resultSet, $recursive, $stack) {
                        
            if (!isset ($resultSet) || !is_array($resultSet)) {
                if (Configure :: read() > 0) {
                    e('<div style = "font: Verdana bold 12px; color: #FF0000">SQL Error in model ' . $model->name . ': ');
                    if (isset ($this->error) && $this->error != null) {
                        e($this->error);
                    }
                    e('</div>');
                }
                return null;
            }
            
            $count = count($resultSet);
            for ($i = 0; $i < $count; $i++) {
                
                $row = & $resultSet[$i];
                $queryData = $this->generateAssociationQuery($model, $linkModel, $type, $association, $assocData, $queryData, $external, $row);
                $fetch = $this->_executeQuery($queryData);
                $fetch = ldap_get_entries($this->database, $fetch);
                $fetch = $this->_ldapFormat($linkModel,$fetch);
                
                if (!empty ($fetch) && is_array($fetch)) {
                        if ($recursive > 0) {
                            foreach ($linkModel->__associations as $type1) {
                                foreach ($linkModel-> {$type1 } as $assoc1 => $assocData1) {
                                    $deepModel = & $linkModel->{$assocData1['className']};
                                    if ($deepModel->alias != $model->name) {
                                        $tmpStack = $stack;
                                        $tmpStack[] = $assoc1;
                                        if ($linkModel->useDbConfig == $deepModel->useDbConfig) {
                                            $db = & $this;
                                        } else {
                                            $db = & ConnectionManager :: getDataSource($deepModel->useDbConfig);
                                        }
                                        $queryData = array();
                                        $db->queryAssociation($linkModel, $deepModel, $type1, $assoc1, $assocData1, $queryData, true, $fetch, $recursive -1, $tmpStack);
                                    }
                                }
                            }
                        }
                    $this->__mergeAssociation($resultSet[$i], $fetch, $association, $type);
    
                } else {
                    $tempArray[0][$association] = false;
                    $this->__mergeAssociation($resultSet[$i], $tempArray, $association, $type);
                }
            }
        }
        
        /**
         * Returns a formatted error message from previous database operation.
         *
         * @return string Error message with error number
         */
        function lastError() {
            if (ldap_errno($this->database)) {
                return ldap_errno($this->database) . ': ' . ldap_error($this->database);
            }
            return null;
        }
    
        /**
         * Returns number of rows in previous resultset. If no previous resultset exists,
         * this returns false.
         *
         * @return int Number of rows in resultset
         */
        function lastNumRows() {
            if ($this->_result and is_resource($this->_result)) {
                return @ ldap_count_entries($this->database, $this->_result);
            }
            return null;
        }
    
        // Usefull public (static) functions--------------------------------------------    
        /**
         * Convert Active Directory timestamps to unix ones
         * 
         * @param integer $ad_timestamp Active directory timestamp
         * @return integer Unix timestamp
         */
        function convertTimestamp_ADToUnix($ad_timestamp) {
            $epoch_diff = 11644473600; // difference 1601<>1970 in seconds. see reference URL
            $date_timestamp = $ad_timestamp * 0.0000001;
            $unix_timestamp = $date_timestamp - $epoch_diff;
            return $unix_timestamp;
        }// convertTimestamp_ADToUnix
        
        /* The following was kindly "borrowed" from the excellent phpldapadmin project */
        function __getLDAPschema() {
            $schemaTypes = array( 'objectclasses', 'attributetypes' );
            $check = @ldap_read($this->database, 'cn=Schema', 'objectClass=*');
            if(ldap_count_entries($this->database, $check) > 0){
            	$schemaDN = 'cn=Schema';
            }else{
            	$schemaDN = 'cn=SubSchema';
            }
            foreach (array('(objectClass=*)','(objectClass=subschema)') as $schema_filter) {
                $this->results = @ldap_read($this->database, $schemaDN, $schema_filter, $schemaTypes,0,0,0,LDAP_DEREF_ALWAYS);
                
    
                if( is_null( $this->results ) ) {
                    $this->log( "LDAP schema filter $schema_filter is invalid!", 'ldap.error');
                    continue;
                }
                
                $schema_entries = @ldap_get_entries( $this->database, $this->results );
    
                
                if ( is_array( $schema_entries ) && isset( $schema_entries['count'] ) ) {
                    break;
                }
                
                unset( $schema_entries );
                $schema_search = null;
            }
     
               if( $schema_entries ) {
                   $return = array();
                   foreach( $schemaTypes as $n ) {
                    $schemaTypeEntries = $schema_entries[0][$n];
                    for( $x = 0; $x < $schemaTypeEntries['count']; $x++ ) {
                        $entry = array();
                        $strings = preg_split('/[\s,]+/', $schemaTypeEntries[$x], -1, PREG_SPLIT_DELIM_CAPTURE);
                        $str_count = count( $strings );
                        for ( $i=0; $i < $str_count; $i++ ) {
                            switch ($strings[$i]) {
                                case '(':
                                    break;
                                case 'NAME':
                                    if ( $strings[$i+1] != '(' ) {
                                        do {
                                            $i++;
                                                if( !isset( $entry['name'] ) || strlen( $entry['name'] ) == 0 )
                                                    $entry['name'] = $strings[$i];
                                                else
                                                    $entry['name'] .= ' '.$strings[$i];
                                        } while ( !preg_match('/\'$/s', $strings[$i]));
                                    } else {
                                        $i++;
                                        do {
                                            $i++;
                                            if( !isset( $entry['name'] ) || strlen( $entry['name'] ) == 0)
                                                $entry['name'] = $strings[$i];
                                            else
                                                $entry['name'] .= ' ' . $strings[$i];
                                        } while ( !preg_match( '/\'$/s', $strings[$i] ) );
                                        do {
                                            $i++;
                                        } while ( !preg_match( '/\)+\)?/', $strings[$i] ) );
                                    }
        
                                    $entry['name'] = preg_replace('/^\'/', '', $entry['name'] );
                                    $entry['name'] = preg_replace('/\'$/', '', $entry['name'] );
                                    break;
                                case 'DESC':
                                    do {
                                        $i++;
                                        if ( !isset( $entry['description'] ) || strlen( $entry['description'] ) == 0 )
                                            $entry['description'] = $strings[$i];
                                        else
                                            $entry['description'] .= ' ' . $strings[$i];
                                    } while ( !preg_match( '/\'$/s', $strings[$i] ) );
                                    break;
                                case 'OBSOLETE':
                                    $entry['is_obsolete'] = TRUE;
                                    break;
                                case 'SUP':
                                    $entry['sup_classes'] = array();
                                    if ( $strings[$i+1] != '(' ) {
                                        $i++;
                                        array_push( $entry['sup_classes'], preg_replace( "/'/", '', $strings[$i] ) );
                                    } else {
                                        $i++;
                                        do {
                                            $i++;
                                            if ( $strings[$i] != '$' )
                                                array_push( $entry['sup_classes'], preg_replace( "/'/", '', $strings[$i] ) );
                                        } while (! preg_match('/\)+\)?/',$strings[$i+1]));
                                    }
                                    break;
                                case 'ABSTRACT':
                                    $entry['type'] = 'abstract';
                                    break;
                                case 'STRUCTURAL':
                                    $entry['type'] = 'structural';
                                    break;
                                case 'SINGLE-VALUE':
                                    $entry['multiValue'] = 'false';
                                    break;
                                case 'AUXILIARY':
                                    $entry['type'] = 'auxiliary';
                                    break;
                                case 'MUST':
                                    $entry['must'] = array();
                                    $i = $this->_parse_list(++$i, $strings, $entry['must']);
    
                                    break;
    
                                case 'MAY':
                                    $entry['may'] = array();
                                    $i = $this->_parse_list(++$i, $strings, $entry['may']);
    
                                    break;
                                default:
                                    if( preg_match( '/[\d\.]+/i', $strings[$i]) && $i == 1 ) {
                                        $entry['oid'] = $strings[$i];
                                    }
                                    break;
                            }
                        }
                        if( !isset( $return[$n] ) || !is_array( $return[$n] ) ) {
                            $return[$n] = array();
                        }
    				//make lowercase for consistency
    		    		$return[strtolower($n)][strtolower($entry['name'])] = $entry;
                        //array_push( $return[$n][$entry['name']], $entry );
                    }
                }
            }
    
            return $return;
        }
    
        function _parse_list( $i, $strings, &$attrs ) {
            /**
             ** A list starts with a ( followed by a list of attributes separated by $ terminated by )
             ** The first token can therefore be a ( or a (NAME or a (NAME)
             ** The last token can therefore be a ) or NAME)
             ** The last token may be terminate by more than one bracket
             */
            $string = $strings[$i];
            if (!preg_match('/^\(/',$string)) {
                // A bareword only - can be terminated by a ) if the last item
                if (preg_match('/\)+$/',$string))
                        $string = preg_replace('/\)+$/','',$string);
    
                array_push($attrs, $string);
            } elseif (preg_match('/^\(.*\)$/',$string)) {
                $string = preg_replace('/^\(/','',$string);
                $string = preg_replace('/\)+$/','',$string);
                array_push($attrs, $string);
            } else {
                // Handle the opening cases first
                if ($string == '(') {
                        $i++;
    
                } elseif (preg_match('/^\(./',$string)) {
                        $string = preg_replace('/^\(/','',$string);
                        array_push ($attrs, $string);
                        $i++;
                }
    
                // Token is either a name, a $ or a ')'
                // NAME can be terminated by one or more ')'
                while (! preg_match('/\)+$/',$strings[$i])) {
                        $string = $strings[$i];
                        if ($string == '$') {
                                $i++;
                                continue;
                        }
    
                        if (preg_match('/\)$/',$string)) {
                                $string = preg_replace('/\)+$/','',$string);
                        } else {
                                $i++;
                        }
                        array_push ($attrs, $string);
                }
            }
            sort($attrs);
    
            return $i;
        }
    
        /**
         * Function not supported
         */
        function execute($query) {
            return null;
        }
        
        /**
         * Function not supported
         */
        function fetchAll($query, $cache = true) {
            return array();
        }
        
        // Logs --------------------------------------------------------------
        /**
         * Log given LDAP query.
         *
         * @param string $query LDAP statement
         * @todo: Add hook to log errors instead of returning false
         */
        function logQuery($query) {
            $this->_queriesCnt++;
            $this->_queriesTime += $this->took;
            $this->_queriesLog[] = array (
                'query' => $query,
                'error' => $this->error,
                'affected' => $this->affected,
                'numRows' => $this->numRows,
                'took' => $this->took
            );
            if (count($this->_queriesLog) > $this->_queriesLogMax) {
                array_pop($this->_queriesLog);
            }
            if ($this->error) {
                return false;
            }
        }
        
        /**
         * Outputs the contents of the queries log.
         *
         * @param boolean $sorted
         */
        function showLog($sorted = false) {
            if ($sorted) {
                $log = sortByKey($this->_queriesLog, 'took', 'desc', SORT_NUMERIC);
            } else {
                $log = $this->_queriesLog;
            }
    
            if ($this->_queriesCnt > 1) {
                $text = 'queries';
            } else {
                $text = 'query';
            }
    
            if (php_sapi_name() != 'cli') {
                print ("<table id=\"cakeSqlLog\" cellspacing=\"0\" border = \"0\">\n<caption>{$this->_queriesCnt} {$text} took {$this->_queriesTime} ms</caption>\n");
                print ("<thead>\n<tr><th>Nr</th><th>Query</th><th>Error</th><th>Affected</th><th>Num. rows</th><th>Took (ms)</th></tr>\n</thead>\n<tbody>\n");
    
                foreach ($log as $k => $i) {
                    print ("<tr><td>" . ($k +1) . "</td><td>{$i['query']}</td><td>{$i['error']}</td><td style = \"text-align: right\">{$i['affected']}</td><td style = \"text-align: right\">{$i['numRows']}</td><td style = \"text-align: right\">{$i['took']}</td></tr>\n");
                }
                print ("</table>\n");
            } else {
                foreach ($log as $k => $i) {
                    print (($k +1) . ". {$i['query']} {$i['error']}\n");
                }
            }
        }
    
        /**
         * Output information about a LDAP query. The query, number of rows in resultset,
         * and execution time in microseconds. If the query fails, an error is output instead.
         *
         * @param string $query Query to show information on.
         */
        function showQuery($query) {
            $error = $this->error;
            if (strlen($query) > 200 && !$this->fullDebug) {
                $query = substr($query, 0, 200) . '[...]';
            }
    
            if ($this->debug || $error) {
                print ("<p style = \"text-align:left\"><b>Query:</b> {$query} <small>[Aff:{$this->affected} Num:{$this->numRows} Took:{$this->took}ms]</small>");
                if ($error) {
                    print ("<br /><span style = \"color:Red;text-align:left\"><b>ERROR:</b> {$this->error}</span>");
                }
                print ('</p>');
            }
        }
        
        // _ private --------------------------------------------------------------
        function _conditions($conditions, $model) {
            $res = '';
            $key = $model->primaryKey;
            $name = $model->name;
    
    	if(is_array($conditions) && count($conditions) == 1) {
    		
    		$sqlHack = "$name.$key";
    		$conditions = str_ireplace($sqlHack, $key, $conditions);
    		foreach($conditions as $k => $v){
    			if($k == $name.'.dn'){
    				$res = substr($v, 0, strpos($v, ','));
    			}elseif(($k == $sqlHack) && ( (empty($v))||($v =='*') ) ){
    				$res = 'objectclass=*';
    			}elseif($k == $sqlHack){
    				$res = "$key=$v";
    			}else{
    				$res = "$k=$v";
    			}
    		}
    		$conditions = $res;
    	}
    
            if (is_array($conditions)) {
                // Conditions expressed as an array 
                if (empty($conditions)){
                    $res = 'objectclass=*';
                }
            }
    
    	if(empty($conditions) ) {
    		$res = 'objectclass=*';
    	}else{
    		$res = $conditions;
    	}
            return $res;
        }
        /**
         * Convert an array into a ldap condition string
         * 
         * @param array $conditions condition 
         * @return string 
         */
        function __conditionsArrayToString($conditions) {
            $ops_rec = array ( 'and' => array('prefix'=>'&'), 'or' => array('prefix'=>'|'));
            $ops_neg = array ( 'and not' => array() , 'or not' => array(), 'not equals' => array());
            $ops_ter = array ( 'equals' => array('null'=>'*'));
            
            $ops = array_merge($ops_rec,$ops_neg, $ops_ter);
            
            if (is_array($conditions)) {
                
                $operand = array_keys($conditions);
                $operand = $operand[0];
                
                if (!in_array($operand,array_keys($ops)) ){
    		$this->log("No operators defined in LDAP search conditions.",'ldap.error');
                    return null;
    	    }
                
                $children = $conditions[$operand];
                
                if (in_array($operand, array_keys($ops_rec)) ) {
                    if (!is_array($children))
                        return null;
                
                    $tmp = '('.$ops_rec[$operand]['prefix'];
                    foreach ($children as $key => $value)  {
                        $child = array ($key => $value);
                        $tmp .= $this->__conditionsArrayToString($child);
                    }
                    return $tmp.')';
                    
                } else if (in_array($operand, array_keys($ops_neg)) ) {
                        if (!is_array($children))
                            return null;
                            
                        $next_operand = trim(str_replace('not', '', $operand));
                        
                        return '(!'.$this->__conditionsArrayToString(array ($next_operand => $children)).')';
                        
                } else if (in_array($operand,  array_keys($ops_ter)) ){
                        $tmp = '';
                        foreach ($children as $key => $value) {
                            if ( !is_array($value) )
                                $tmp .= '('.$key .'='.((is_null($value))?$ops_ter['equals']['null']:$value).')';
                            else
                                foreach ($value as $subvalue) 
                                    $tmp .= $this->__conditionsArrayToString(array('equals' => array($key => $subvalue)));
                        }
                        return $tmp;
                }            
            }
        }
    
        function checkBaseDn( $targetDN ){
    	$parts = preg_split('/,\s*/', $this->config['basedn']);
    	$pattern = '/'.implode(',\s*', $parts).'/i';
    	return(preg_match($pattern, $targetDN));
        }
        
        function _executeQuery($queryData = array (), $cache = true){
        	$t = getMicrotime();
        	
    	$pattern = '/,[ \t]+(\w+)=/';
    	$queryData['targetDn'] = preg_replace($pattern, ',$1=',$queryData['targetDn']);	
            if($this->checkBaseDn($queryData['targetDn']) == 0){
    		$this->log("Missing BaseDN in ". $queryData['targetDn'],'debug');
                
            	if($queryData['targetDn'] != null){
            		$seperator = (substr($queryData['targetDn'], -1) == ',') ? '' : ',';
    				if( (strpos($queryData['targetDn'], '=') === false) && (isset($this->model) && !empty($this->model)) ){
    					//Fix TargetDN here 
    					$key = $this->model->primaryKey;
    					$table = $this->model->useTable;
    					$queryData['targetDn'] = $key.'='.$queryData['targetDn'].', '.$table.$seperator.$this->config['basedn'];
    				}else{
    					$queryData['targetDn'] = $queryData['targetDn'].$seperator.$this->config['basedn'];
    				}
            	}else{
            		$queryData['targetDn'] = $this->config['basedn'];
            	}
            }
            
            $query = $this->_queryToString($queryData);
            if ($cache && isset ($this->_queryCache[$query])) {
                if (strpos(trim(strtolower($query)), $queryData['type']) !== false) {
                    $res = $this->_queryCache[$query];
                }
            } else {
            	
                switch ($queryData['type']) {
                    case 'search':
                        // TODO pb ldap_search & $queryData['limit']
    		    if( empty($queryData['fields']) ){
    			$queryData['fields'] = $this->defaultNSAttributes();
    		    }
                        
                    	//Handle LDAP Scope
                        if(isset($queryData['scope']) && $queryData['scope'] == 'base'){
                        	$res = @ ldap_read($this->database, $queryData['targetDn'], $queryData['conditions'], $queryData['fields']);
                        }elseif(isset($queryData['scope']) && $queryData['scope'] == 'one'){
                        	$res = @ ldap_list($this->database, $queryData['targetDn'], $queryData['conditions'], $queryData['fields']);
                        }else{
                        	if($queryData['fields'] == 1) $queryData['fields'] = array(); 
                        	$res = @ ldap_search($this->database, $queryData['targetDn'], $queryData['conditions'], $queryData['fields'], 0, $queryData['limit']);
                        }
    					
                        if(!$res){
                            $res = false;
    						$errMsg = ldap_error($this->database);
                            $this->log("Query Params Failed:".print_r($queryData,true).' Error: '.$errMsg,'ldap.error');
                            $this->count = 0;
                        }else{
                        	$this->count = ldap_count_entries($this->database, $res);
                        }
                        
                        if ($cache) {
                        	if (strpos(trim(strtolower($query)), $queryData['type']) !== false) {
                        		$this->_queryCache[$query] = $res;
                            }
                        }
                        break;
                    case 'delete':
                        $res = @ ldap_delete($this->database, $queryData['targetDn'] . ',' . $this->config['basedn']);             
                        break;
                    default:
                        $res = false;
                        break;
                }
            }
                    
            $this->_result = $res;
            $this->took = round((getMicrotime() - $t) * 1000, 0);
            $this->error = $this->lastError();
            $this->numRows = $this->lastNumRows();
    
            if ($this->fullDebug) {
                $this->logQuery($query);
            }
    
            return $this->_result;
        }
        
        function _queryToString($queryData) {
            $tmp = '';
            if (!empty($queryData['scope'])) 
                $tmp .= ' | scope: '.$queryData['scope'].' ';
    
            if (!empty($queryData['conditions'])) 
                $tmp .= ' | cond: '.$queryData['conditions'].' ';
    
            if (!empty($queryData['targetDn'])) 
                $tmp .= ' | targetDn: '.$queryData['targetDn'].' ';
    
            $fields = '';
            if (!empty($queryData['fields']) && is_array( $queryData['fields'] ) ) {
    			$fields = implode(', ', $queryData['fields']);
                $tmp .= ' |fields: '.$fields.' ';
            }
        
            if (!empty($queryData['order']))         
                $tmp .= ' | order: '.$queryData['order'][0].' ';
    
            if (!empty($queryData['limit']))
                $tmp .= ' | limit: '.$queryData['limit'];
    
            return $queryData['type'] . $tmp;
        }
    
        function _ldapFormat(& $model, $data) {
            $res = array ();
    
            foreach ($data as $key => $row){
                if ($key === 'count')
                    continue;
        
                foreach ($row as $key1 => $param){
                    if ($key1 === 'dn') {
                        $res[$key][$model->name][$key1] = $param;
                        continue;
                    }
                    if (!is_numeric($key1))
                        continue;
                    if ($row[$param]['count'] === 1)
                        $res[$key][$model->name][$param] = $row[$param][0];
                    else {
                        foreach ($row[$param] as $key2 => $item) {
                            if ($key2 === 'count')
                                continue;
                            $res[$key][$model->name][$param][] = $item;
                        }
                    }
                }
            }
            return $res;
        }
        
        function _ldapQuote($str) {
            return str_replace(
                    array( '\\', ' ', '*', '(', ')' ),
                    array( '\\5c', '\\20', '\\2a', '\\28', '\\29' ),
                    $str
            );
        }
        
        // __ -----------------------------------------------------
        function __mergeAssociation(& $data, $merge, $association, $type) {
                    
            if (isset ($merge[0]) && !isset ($merge[0][$association])) {
                $association = Inflector :: pluralize($association);
            }
    
            if ($type == 'belongsTo' || $type == 'hasOne') {
                if (isset ($merge[$association])) {
                    $data[$association] = $merge[$association][0];
                } else {
                    if (count($merge[0][$association]) > 1) {
                        foreach ($merge[0] as $assoc => $data2) {
                            if ($assoc != $association) {
                                $merge[0][$association][$assoc] = $data2;
                            }
                        }
                    }
                    if (!isset ($data[$association])) {
                        $data[$association] = $merge[0][$association];
                    } else {
                        if (is_array($merge[0][$association])) {
                            $data[$association] = array_merge($merge[0][$association], $data[$association]);
                        }
                    }
                }
            } else {
                if ($merge[0][$association] === false) {
                    if (!isset ($data[$association])) {
                        $data[$association] = array ();
                    }
                } else {
                    foreach ($merge as $i => $row) {
                        if (count($row) == 1) {
                            $data[$association][] = $row[$association];
                        } else {
                            $tmp = array_merge($row[$association], $row);
                            unset ($tmp[$association]);
                            $data[$association][] = $tmp;
                        }
                    }
                }
            }
        }
        
        /**
         * Private helper method to remove query metadata in given data array.
         *
         * @param array $data
         */
        function __scrubQueryData(& $data) {
            if (!isset ($data['type']))
                $data['type'] = 'default';
            
            if (!isset ($data['conditions'])) 
                $data['conditions'] = array();
    
            if (!isset ($data['targetDn'])) 
                $data['targetDn'] = null;
        
            if (!isset ($data['fields']) && empty($data['fields'])) 
                $data['fields'] = array ();
            
            if (!isset ($data['order']) && empty($data['order'])) 
                $data['order'] = array ();
    
            if (!isset ($data['limit']))
                $data['limit'] = null;
        }
        
        function __getObjectclasses() {
            $cache = null;
            if ($this->cacheSources !== false) {
                if (isset($this->__descriptions['ldap_objectclasses'])) {
                    $cache = $this->__descriptions['ldap_objectclasses'];
                } else {
                    $cache = $this->__cacheDescription('objectclasses');
                }
            }
                            
            if ($cache != null) {
                return $cache;
            }
            
            // If we get this far, then we haven't cached the attribute types, yet!
            $ldapschema = $this->__getLDAPschema();
            $objectclasses = $ldapschema['objectclasses'];
            
            // Cache away
            $this->__cacheDescription( 'objectclasses', $objectclasses );
            
            return $objectclasses;
        }
        
        function boolean() {
            return null;
        }
    
    /**
     * Returns the count of records
     *
     * @param model $model
     * @param string $func Lowercase name of SQL function, i.e. 'count' or 'max'
     * @param array $params Function parameters (any values must be quoted manually)
     * @return string       entry count
     * @access public
     */
            function calculate(&$model, $func, $params = array()) {
                    $params = (array)$params;
    
                    switch (strtolower($func)) {
                            case 'count':
    							if(empty($params) && $model->id){
    								//quick search to make sure it exsits
    								$queryData['targetDn'] = $model->id;
    								$queryData['conditions'] = 'objectClass=*';
    								$queryData['scope'] = 'base';
    								$query = $this->read($model, $queryData);
    							}
    							return $this->count;
    							break; 
                            case 'max':
                            case 'min':
                            break;
                    }
            }
    
    	function describe(&$model, $field = null){
    		$schemas = $this->__getLDAPschema();
    		$attrs = $schemas['attributetypes'];
    		ksort($attrs);
    		if(!empty($field)){
    			return($attrs[strtolower($field)]);
    		}else{
    			return $attrs;
    		}
    	}
    
    	function in_arrayi( $needle, $haystack ) {
    		$found = false;
    		foreach( $haystack as $attr => $value ) {
    		    if( strtolower( $attr ) == strtolower( $needle ) ) {
    			$found = true;
    		    }
    		    elseif( strtolower( $value ) == strtolower( $needle ) ) {
    			$found = true;
    		    }
    		}   
    		return $found;
    	} 
    
    /**
    * If you want to pull everything from a netscape stype ldap server 
    * iPlanet, Redhat-DS, Project-389 etc you need to ask for specific 
    * attributes like so.  Other wise the attributes listed below wont
    * show up
    */
    	function defaultNSAttributes(){
    		$fields = '* accountUnlockTime aci copiedFrom copyingFrom createTimestamp creatorsName dncomp entrydn entryid hasSubordinates ldapSchemas ldapSyntaxes modifiersName modifyTimestamp nsAccountLock nsAIMStatusGraphic nsAIMStatusText nsBackendSuffix nscpEntryDN nsds5ReplConflict nsICQStatusGraphic nsICQStatusText nsIdleTimeout nsLookThroughLimit nsRole nsRoleDN nsSchemaCSN nsSizeLimit nsTimeLimit nsUniqueId nsYIMStatusGraphic nsYIMStatusText numSubordinates parentid passwordAllowChangeTime passwordExpirationTime passwordExpWarned passwordGraceUserTime passwordHistory passwordRetryCount pwdExpirationWarned pwdGraceUserTime pwdHistory pwdpolicysubentry retryCountResetTime subschemaSubentry';
    		return(explode(' ', $fields));
    	}
    
    } // LdapSource
    ?>



So lets dive right in below is the database config we will use.

::

    
    class DATABASE_CONFIG {
    
    	// if using ssl set 'host' => ldaps://hostname and 'port' => 636
            // If using tls set 'tls' => true and 'port' => 389
    
    
    	var $ldap = array (
    		'datasource' => 'ldap',
    		'host' => array( 'ldap.example.com', 'ldap2.example.com'),                                        
    		'basedn' => 'dc=examnple,dc=com',
    		'login' => '', 
    		'password' => '',                
    		'database' => '',
                    'tls'         => false,
    		'version' => 3                    
    	);     
    }

You notice that the variables database, login and password are blank.
Keep at least database this way. You can populate login and password
if don't want your ldap connections to be anonymous. I keep mine blank
because I have written my own auth component that uses ldap, So once
I'm authed that gets passed to the datasource instead. This is a ugly
hack that I've written another `post about`_.

You may also notice that host is an array. In most LDAP environments
you will have multiple LDAP servers for redundancy and load balance.
This makes sure you can continue to manage your system if one of them
goes down. You don't have to list two servers here. This can just be
single LDAP URI

Here is our Person model for accessing the users in your LDAP tree.


Model Class:
````````````

::

    <?php  
    class Person extends AppModel {
    
    	var $name = 'Person';
        
    	var $useDbConfig = 'ldap';
    
    	// This would be the ldap equivalent to a primary key if your dn is 
    	// in the format of uid=username, ou=people, dc=example, dc=com
    	var $primaryKey = 'uid';     
    
    	// The table would be the branch of your basedn that you defined in 
    	// the database config
    	var $useTable = 'ou=people'; 
    
    	var $validate = array(
    		'cn' => array(
    			'alphaNumeric' => array
    				'rule' => array('custom', '/^[a-zA-Z]*$/'),
    				'required' => true,
    				'on' => 'create',
    				'message' => 'Only Letters and Numbers can be used for Display Name.'
    			),
    			'between' => array(
    				'rule' => array('between', 5, 15),
    				'on' => 'create',
    				'message' => 'Between 5 to 15 characters'
    			)
            ),
            'sn' => array(
    			'rule' => array('custom', '/^[a-zA-Z]*$/'),
    			'required' => true,
    			'on' => 'create',
    			'message' => 'Only Letters and Numbers can be used for Last Name.'
            ),
            'userpassword' => array(
    			'rule' => array('minLength', '8'),
    			'message' => 'Mimimum 8 characters long.'
            ),
            'email' => array(
    			'rule' => 'email',
    			'required' => true,
    			'on' => 'create',
    			'message' => 'Must Contain a Valid Email Address.'
    		),
            'uid' => array(
    			'rule' => 'alphaNumeric',
    			'required' => true,
    			'on' => 'create',
    			'message' => 'Only Letters and Numbers can be used for Username.'
            ),
        );
    
    			
    }
    ?>

Here is a very basic controller to accompany our people model. It
demonstrates the important core functions and should get you started
on using this data source with your own application.


Controller Class:
`````````````````

::

    <?php 
    class PeopleController extends AppController {
    
    	var $name = 'People';    
    	var $components = array('RequestHandler');
    	var $helpers = array('Form','Html','Javascript', 'Ajax');
    
     
    	function add(){
                if(!empty($this->data)){
    			$this->data['Person']['objectclass'] = array('top', 'organizationalperson', 'inetorgperson','person','posixaccount','shadowaccount');
    
    			if($this->data['Person']['password'] == $this->data['Person']['password_confirm']){
    				$this->data['userpassword'] = $this->data['Person']['password'];
    				unset($this->data['Person']['password']);
    				unset($this->data['Person']['password_confirm']);
    			
    				if(!isset($this->data['Person']['homedirectory'])&& isset($this->data['Person']['uid'])){
    					$this->data['Person']['homedirectory'] = '/home/'.$this->data['Person']['uid'];
    				}
    
    				if ($this->People->save($this->data)) {
    					$this->Session->setFlash('People Was Successfully Created.');
    					$id = $this->People->id;
    					$this->redirect(array('action' => 'view', 'id'=> $id));
    				}else{
    					$this->Session->setFlash("People couldn't be created.");
    				}
    			}else{
    				$this->Session->setFlash("Passwords don't match.");
    			}
                    }
    		$this->layout = 'people';
    	}
    	
    	function view( $id ){
    		if(!empty($id)){
    			$filter = $this->People->primaryKey."=".$id;
    			$people = $this->People->find('first', array( 'conditions'=>$filter));
    			$this->set(compact('people'));
    		}
    		$this->layout = 'people';
    	}
    
    	function delete($id = null) {
    		$this->People->id = $id;
    		return $this->People->del($id);
    	}
    
    }
    ?>

Note above, I just made up a list of objectclasses I wanted my user to
have. For a Unix user, those would be the only objectclasses you
really need. If you are doing more or something else you will need to
place your objectclasses here and also modify your add view form to
reflect.


So lets talk about somethings here, in our model we define $primaryKey
& $useTable variables. The $useTable is the branch of the ldap server.
For this models purpose we define our table as **'ou=people'**. This
makes sure that objects we create (I.E. Users/people) will be added
under the organization unit people. It also makes sure that when you
pass something like 'jdoe' to the delete action it will search that
branch for the user object to delete. The $primaryKey also helps in
the creation and deleting of users. It makes sure that the dn is
created as uid, this is helpful to make sure that a user doesn't
already have that user name. Also since ldap is case insensitive you
don't have to worry about the possible variations of the object names
when checking the existence.

I didn't really show any views here, because their is nothing special
in the views. You create them and use them like any other data source.
Basically what ever you set in your controller actions will be
available in your view.

Now your model isn't limited to one branch or object type. If you
wanted to create a browser for example your could define a model like
the following.


Model Class:
````````````

::

    <?php 
    class Browser extends AppModel {
        var $name = 'Browser';
        var $useDbConfig = 'ldap';
        var $primaryKey = 'dn';
        var $useTable = '';
    }
    ?>

You'll notice here we set our $useTable to nothing (important, you get
errors about no db defined from CakePHP if this missing). The really
interesting part here is that we set $primaryKey to dn. This is the
ultimate primary key for our type of data source. The difference here
is that when we create/delete an object we have to pass it the full
dn.

Our new data source also adds some new options to the find function.
$options['targetDN'] This is more like the point in the tree we want
to start our search. If you don't define it, then it defaults to the
$useTable.$config[$useDbConfig]['basedn'] if your $useTable variable
is empty it defaults to the basedn configured in your database config.

$options['scope'] If you've worked with ldap before then you are
familiar with the concept of search scopes. You have three search
scopes, 'sub', 'one, & 'base'. Basically sub means search from this
point down the tree. one means search one level below this point and
base means search just this point. For example if you wanted to see if
a user already existed you could set the targetDn to
uid=jdoe,ou=people,dc=example,dc=com and it will check if this object
already exists. The default scope is sub
Moving forward I want to add checks to this LDAP datasource to check
what the LDAP server being used is and add special functionality to
this data source like checking object permissions and getting implicit
attibutes returned by default for Netscape style LDAP servers.
`1`_|`2`_|`3`_|`4`_


More
````

+ `Page 1`_
+ `Page 2`_
+ `Page 3`_
+ `Page 4`_

.. _post about: http://www.analogrithems.com/rant/2009/06/13/ldapauth-component-for-cakephp/
.. _http://github.com/analogrithems/idbroker/blob/master/models/datasources/ldap_source.php: http://github.com/analogrithems/idbroker/blob/master/models/datasources/ldap_source.php
.. _euphrate: http://bakery.cakephp.org/articles/view/ldap-datasource-for-cakephp
.. _Gservat: http://memdump.wordpress.com/2008/04/26/ldap-data-source-now-with-full-crud/
.. _Page 4: :///articles/view/4caea0e5-0c4c-4b3e-9b4e-4c8c82f0cb67/lang:eng#page-4
.. _Page 1: :///articles/view/4caea0e5-0c4c-4b3e-9b4e-4c8c82f0cb67/lang:eng#page-1
.. _Page 3: :///articles/view/4caea0e5-0c4c-4b3e-9b4e-4c8c82f0cb67/lang:eng#page-3
.. _Page 2: :///articles/view/4caea0e5-0c4c-4b3e-9b4e-4c8c82f0cb67/lang:eng#page-2
.. meta::
    :title: LDAP data source with full CRUD support
    :description: CakePHP Article related to ldap,datasource,Models
    :keywords: ldap,datasource,Models
    :copyright: Copyright 2009 analogrithems
    :category: models

