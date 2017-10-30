# spring-boot-call-store-procedure
Dynamic calling store procedure and store function in database

Dynamic calling store procedure and function
1. Via Callable Statement
2. Via SimpleJdbcCall spring-boot-starter-jdbc

See Code Snippets

1. Store Procedure
<pre>
DELIMITER $$

USE `training_sp`$$

DROP PROCEDURE IF EXISTS `create_product`$$

CREATE DEFINER=`root`@`localhost` PROCEDURE `create_product`(id VARCHAR(255), p_code VARCHAR(255),p_name VARCHAR(255),weight BIGINT)
BEGIN
	
	INSERT INTO product(id, CODE,NAME,weight) VALUES(id,p_code,p_name,weight);
    END$$

DELIMITER ;
</pre>

2. Function
<pre>
DELIMITER $$

USE `training_sp`$$

DROP FUNCTION IF EXISTS `count_product`$$

CREATE DEFINER=`root`@`localhost` FUNCTION `count_product`() RETURNS BIGINT(20)
BEGIN
	DECLARE v_count BIGINT DEFAULT 0;
    
	SELECT  COUNT(1) INTO v_count FROM product;
	RETURN v_count;
    END$$

DELIMITER ;
</pre>

3. Implementation Using SimpleJdbcCall

<pre>
@Component
public class JdbcTemplateUtils {
    Logger logger = LoggerFactory.getLogger(JdbcTemplateUtils.class);
    private SimpleJdbcCall simpleJdbcCall;

    @Autowired
    @Qualifier("data_mysql")
    public void setDatasource(DataSource datasource){
        this.simpleJdbcCall =new SimpleJdbcCall(datasource);
    }
    public void callStoreProcedure(String procedureName, Map<String,Object> parameters){
        simpleJdbcCall.withProcedureName(procedureName);
        MapSqlParameterSource inParams = new MapSqlParameterSource();
        if(null!=parameters) {
            for (Map.Entry<String, Object> parameter : parameters.entrySet()) {
                inParams.addValue(parameter.getKey(), parameter.getValue());
            }
        }
        simpleJdbcCall.execute(inParams);
        logger.info("PROCEDURE {} IS CALLED",procedureName);
    }

    public Object callStoredFunction(String functionName, Map<String,Object> parameters, Class<?> classreturn){
        simpleJdbcCall.withFunctionName(functionName);
        simpleJdbcCall.withReturnValue();
        MapSqlParameterSource inParams = new MapSqlParameterSource();
        if(null!=parameters) {
            for (Map.Entry<String, Object> parameter : parameters.entrySet()) {
                inParams.addValue(parameter.getKey(), parameter.getValue());
            }
        }
        logger.info("FUNCTION {} IS CALLED",functionName);
        return simpleJdbcCall.executeFunction(classreturn,inParams);
    }

}
</pre>

3.1 Calling the store procedure

<pre>
        Map<String,Object> params=new HashMap<>();
        params.put("id",UUID.randomUUID().toString());
        params.put("p_code",product.getCode());
        params.put("p_name",product.getName());
        params.put("weight",product.getWeight());
        jdbcUtils.callStoreProcedure("create_product",params);
</pre>

3.2 Calling the Stored function
<pre>
    Long count=(Long) jdbcUtils.callStoredFunction( "count_product",null,Long.class); 
</pre>

4. Implementation using CallableStatement

<pre>
/**
 * Created by krisna putra on 10/28/2017.
 */
@Component
public class DatabaseUtils {
    Logger log= LoggerFactory.getLogger(DatabaseUtils.class);
    private final String callFunction  = "{ ? = call #statement}";
    private final String callProcedure = "{ call #statement}";
    CallableStatement callableStatement;

    private DataSource dataSource;

    @Autowired
    @Qualifier("data_mysql")
    public void setDataSource(DataSource dataSource){
        this.dataSource=dataSource;
    }
    public Object callStoredFunction(int sqlReturnType, String functionName, Object[] params){
        try {
            callableStatement= dataSource.getConnection()
                                .prepareCall(
                                        callFunction.replace("#statement",functionName)
                                );

            callableStatement.registerOutParameter(1,sqlReturnType);
            if(params!=null) {
                for (int i = 0; i < params.length; i++) {
                    callableStatement.setObject((i+2),params[i]);
                }
            }
            callableStatement.execute();
            log.info("FUNCTION {} is CALLED",functionName);
            return callableStatement.getObject(1);
        } catch (SQLException e) {
            log.error("Error Call Function {} ",functionName,e);
        }finally {
            try {
                if(callableStatement!=null){
                    callableStatement.close();
                }
            }catch (Exception e2){
                log.error("Error Closed Connection ",e2);
            }
        }
        return  null;
    }
    public void callStoredProcedure( String procedureName, Object[] params){
        try {
            callableStatement= dataSource.getConnection()
                    .prepareCall(
                            callProcedure.replace("#statement",procedureName)
                    );
            if(params!=null) {
                for (int i = 0; i < params.length; i++) {
                    callableStatement.setObject((i+1),params[i]);
                }
            }
            callableStatement.execute();
            log.info("PROCEDURE {} is CALLED",procedureName);
        } catch (SQLException e) {
            log.error("Error Call Procedure {} ",procedureName,e);
        }finally {
            try {
                if(callableStatement!=null){
                    callableStatement.close();
                }
            }catch (Exception e2){
                log.error("Error Closed Connection ",e2);
            }
        }
    }

}
</pre>

4.1 Call the Stored Procedure
<pre>
    Object[] params=new Object[]{
                    UUID.randomUUID().toString(),
                    product.getCode(),
                    product.getName(),
                    product.getWeight()
            };
            utils.callStoredProcedure("create_product(?,?,?,?)",params);
</pre>

4.2 Call the Stored Function
<pre>
 Long count=(Long) utils.callStoredFunction(Types.BIGINT, "count_product()",null);
</pre>

Done.
Happy Coding;
