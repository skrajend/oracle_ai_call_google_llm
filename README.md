# oracle_ai_call_google_llm
Oracle PLSQL to Call Google LLM

# Oracle PL/SQL Procedure to Call Google's Gemini Language Model (LLM)

This repository contains an Oracle PL/SQL procedure that enables you to interact with Google's Gemini language model directly from your Oracle database. This allows you to leverage the power of LLMs for tasks like text generation, summarization, translation, and more, all within your existing Oracle environment.

## Overview

The `oracle_call_google_llm` procedure handles the complexities of making HTTP requests to the Gemini API, managing authentication, and processing the JSON response. It provides a simple interface for sending prompts and receiving the generated text from the model.

## Code Explanation

The main PL/SQL procedure is `oracle_call_google_llm`. Let's break down its key sections:

**1. Procedure Definition:**

```sql
CREATE OR REPLACE PROCEDURE oracle_call_google_llm (
  p_prompt IN VARCHAR2,
  p_result OUT CLOB,
  p_debug IN BOOLEAN DEFAULT FALSE,
  p_config_name IN VARCHAR2 DEFAULT 'default'
) AS
  l_http_request  UTL_HTTP.req;
  l_http_response UTL_HTTP.resp;
  l_text          VARCHAR2(32767);
  l_api_key       VARCHAR2(100);
  l_url           VARCHAR2(1000);
  l_model_name    VARCHAR2(100);
  l_request_body  CLOB;
  l_status        NUMBER;
  l_reason        VARCHAR2(256);
  l_response_text CLOB; --Added
  
  -- Helper procedure for debug output
  PROCEDURE debug_print(p_message IN VARCHAR2) IS
  BEGIN
    IF p_debug THEN
      DBMS_OUTPUT.put_line(p_message);
    END IF;
  END debug_print;
BEGIN
  debug_print('Starting API call procedure');

  -- Retrieve API key and model name from llm_config
  BEGIN
      SELECT api_key, model_name, api_url
      INTO l_api_key, l_model_name, l_url
      FROM llm_config
      WHERE config_name = p_config_name;
  EXCEPTION
      WHEN NO_DATA_FOUND THEN
          RAISE_APPLICATION_ERROR(-20001, 'Configuration not found: ' || p_config_name);
  END;
  
  -- Create the request body in JSON format
  l_request_body := '{
    "contents": [
      {
        "parts": [
          {
            "text": "' || p_prompt || '"
          }
        ]
      }
    ]
  }';
  
  debug_print('Request body created');
  debug_print('URL: ' || l_url || '?key=' || SUBSTR(l_api_key, 1, 5) || '...');
  
  -- Set up the HTTP request
  l_http_request := UTL_HTTP.begin_request(
    url => l_url || '?key=' || l_api_key,
    method => 'POST',
    http_version => 'HTTP/1.1'
  );
  
  -- Set headers
  UTL_HTTP.set_header(l_http_request, 'Content-Type', 'application/json');
  UTL_HTTP.set_header(l_http_request, 'Content-Length', LENGTH(l_request_body));
  
  -- Set body
  UTL_HTTP.write_text(l_http_request, l_request_body);
  
  debug_print('Request sent, waiting for response');
  
  -- Get response
  l_http_response := UTL_HTTP.get_response(l_http_request);
  
  -- Get the status code
  l_status := l_http_response.status_code;
  l_reason := l_http_response.reason_phrase;
  
  debug_print('Response received. Status: ' || l_status || ' ' || l_reason);
  
  -- Process response
  l_response_text := ''; -- Initialize the response text variable
  BEGIN
    LOOP
      UTL_HTTP.read_text(l_http_response, l_text, 32767);
      l_response_text := l_response_text || l_text;
      debug_print('Read chunk of response data');
    END LOOP;
  EXCEPTION
    WHEN UTL_HTTP.end_of_body THEN
      debug_print('Finished reading response body');
      NULL;
  END;
  
  -- Close the connections properly
  UTL_HTTP.end_response(l_http_response);

  -- Extract the text content from the JSON response
  IF (l_status = 200) THEN
      -- Try to parse the JSON response. Adjust this logic based on actual response structure
        BEGIN
            SELECT JSON_VALUE(l_response_text, '$.candidates[0].content.parts[0].text') INTO p_result FROM dual;
        EXCEPTION WHEN OTHERS THEN
            p_result := l_response_text;
            debug_print('Failed to extract text from json response');
        END;
  ELSE
      p_result := 'ERROR: ' || l_reason;
      debug_print('The http request was not successful');
  END IF;
  
  debug_print('Response length: ' || LENGTH(p_result));
  IF p_debug AND LENGTH(p_result) > 0 THEN
    debug_print('First 100 chars: ' || SUBSTR(p_result, 1, 100));
  ELSIF p_debug THEN
    debug_print('Empty response received');
  END IF;
  
EXCEPTION
  WHEN OTHERS THEN
    debug_print('Error: ' || SQLERRM);
    debug_print('Error backtrace: ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
    
    -- Error handling without using is_open
    BEGIN
      UTL_HTTP.end_request(l_http_request);
    EXCEPTION
      WHEN OTHERS THEN
        debug_print('Error closing request: ' || SQLERRM);
    END;
    
    BEGIN
      UTL_HTTP.end_response(l_http_response);
    EXCEPTION
      WHEN OTHERS THEN
        debug_print('Error closing response: ' || SQLERRM);
    END;
    
    p_result := 'ERROR: ' || SQLERRM;
END;
/
```

p_prompt: Input parameter, the text prompt you want to send to the Gemini model.
p_result: Output parameter, a CLOB variable that will store the response from the Gemini model. CLOB is used because responses from LLMs can be very large.
p_debug: Input parameter, a BOOLEAN flag (defaulting to FALSE). If set to TRUE, it will print debug information to the DBMS_OUTPUT stream, aiding in troubleshooting.
p_config_name: Input parameter, the name of the configuration from the llm_config table to use. This allows you to easily switch between different API keys, model names, and endpoints. Defaults to 'default'.

Configuration Table (llm_config):

CREATE TABLE llm_config (
    config_name VARCHAR2(50) PRIMARY KEY,
    api_key VARCHAR2(100) NOT NULL,
    model_name VARCHAR2(100) NOT NULL,
    api_url VARCHAR2(500) NOT NULL
);

INSERT INTO llm_config (config_name, api_key, model_name, api_url)
VALUES ('default', 'YOUR_ACTUAL_API_KEY', 'gemini-2.0-flash', 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent');

Defines the llm_config table, which stores the configuration settings for different LLM models. This is critical for managing API keys, model names, and API endpoints without hardcoding them in the procedure. Replace 'YOUR_ACTUAL_API_KEY' with your actual Google Gemini API key.

Prerequisites
Oracle Database: You'll need access to an Oracle database instance.

UTL_HTTP Package: The UTL_HTTP package must be installed and configured in your Oracle database. This package allows you to make HTTP requests from PL/SQL. You might need to grant necessary privileges to the user executing the procedure:

EXEC DBMS_NETWORK_ACL_ADMIN.CREATE_ACL (
  acl         => 'llm_http_acl.xml',
  description => 'ACL to allow HTTP calls to the LLM API',
  principal   => 'YOUR_ORACLE_USER',  -- Replace with your Oracle username
  is_grant    => TRUE,
  privilege   => 'connect'
);

EXEC DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL (
  acl         => 'llm_http_acl.xml',
  host        => 'generativelanguage.googleapis.com',
  lower_port  => 80,
  upper_port  => 443
);

COMMIT;

Replace YOUR_ORACLE_USER with the actual username of the Oracle user that will be running the PL/SQL procedure. This ensures the user has the necessary permissions to connect to the Gemini API endpoint. You may also need to configure a wallet if your database requires it for HTTPS connections.

Google Gemini API Key: You'll need a valid API key from Google to access the Gemini API. Obtain this from the Google Cloud Console.
Gemini API Enabled: Ensure the Gemini API is enabled in your Google Cloud project.

Usage Examples
1. Call with Debugging Enabled:

SET SERVEROUTPUT ON
DECLARE
    l_result CLOB;
BEGIN
    oracle_call_google_llm('What is the capital of France?', l_result, TRUE);
    DBMS_OUTPUT.put_line(l_result);
END;
/

This example calls the oracle_call_google_llm procedure with the prompt "What is the capital of France?" and sets the p_debug parameter to TRUE. The output, including debug messages, will be displayed in the DBMS_OUTPUT stream.

2. Call with Debugging Disabled (Default):

SET SERVEROUTPUT ON
DECLARE
    l_result CLOB;
BEGIN
    oracle_call_google_llm('Summarize this text:  The quick brown fox jumps over the lazy dog.', l_result);
    DBMS_OUTPUT.put_line(l_result);
END;
/

3. Call with a specific Configuration:

SET SERVEROUTPUT ON
DECLARE
    l_result CLOB;
BEGIN
    oracle_call_google_llm('Translate "Hello World" to Spanish.', l_result, FALSE, 'gemini-2.0-flash');
    DBMS_OUTPUT.put_line(l_result);
END;
/

This example calls the procedure using the configuration named 'gemini-pro'. This is important if you need to use a different model or a different API key for specific requests. Make sure you have properly configured the llm_config table with the 'gemini-pro' entry.

Customization
Prompt Engineering: Experiment with different prompts to achieve the desired results from the Gemini model. The quality of the prompt significantly impacts the response.

Error Handling: Enhance the error handling to log errors to a table or send notifications.

Security: Implement more robust security measures for storing and managing API keys (e.g., using Oracle Wallet). Consider passing the API Key via a header rather than in the URL.

Response Parsing: Adapt the JSON parsing logic to handle different response structures from the Gemini API. The provided code assumes a specific response format, and you might need to adjust it based on the specific API endpoint and model you are using.

Rate Limiting: Implement rate limiting to avoid exceeding the API usage limits.

Configuration: Add more parameters to the llm_config table to control other aspects of the API call, such as temperature, top_p, and max_tokens. You can then pass these parameters in the request body.

Asynchronous Processing: For long-running requests, consider using asynchronous processing to avoid blocking the database session.

UTL_HTTP Limitations: UTL_HTTP has some limitations (older protocol support, complexity). Consider using APEX_WEB_SERVICE (if available) for a potentially simpler and more modern approach to HTTP requests. However, UTL_HTTP is widely available across Oracle versions, making it a more portable solution.

Potential Issues and Troubleshooting
UTL_HTTP Not Installed/Configured: Ensure the UTL_HTTP package is properly installed and configured. Grant necessary privileges to the Oracle user.

Network Connectivity: Verify that the Oracle database server can connect to the internet and specifically to the Gemini API endpoint. Firewall rules might be blocking the connection.

Invalid API Key: Double-check that the API key in the llm_config table is correct and has not expired.

Incorrect API Endpoint: Ensure that the API URL in the llm_config table is the correct endpoint for the Gemini API.

JSON Parsing Errors: If the JSON parsing fails, examine the raw response from the Gemini API to understand its structure and adjust the JSON path accordingly. Use a JSON validator to check the validity of the response.

Rate Limiting: If you encounter rate limiting errors, implement rate limiting logic in your PL/SQL procedure. Consult the Gemini API documentation for rate limit details.

ORA-29273: HTTP client error: SSL handshake failed: This common error often relates to missing or misconfigured Oracle Wallet for SSL/TLS connections. Follow Oracle's documentation to properly configure Oracle Wallet for HTTPS connections if you are using HTTPS.

Permissions Errors: Ensure that the Oracle user running the procedure has the necessary permissions to access the llm_config table and execute the UTL_HTTP package.

Long responses getting truncated: If the LLM generates very long responses exceeding PL/SQL string limits, experiment with smaller chunk sizes in UTL_HTTP.read_text or consider streaming the response directly to a file instead of storing it in a CLOB in memory first.

Security Considerations
API Key Protection: Never hardcode API keys directly into your PL/SQL code. Store them securely, such as in the llm_config table or an Oracle Wallet. Restrict access to the llm_config table.

Input Validation: Sanitize and validate user inputs to prevent prompt injection attacks. Be cautious about directly incorporating user-provided data into the prompt without proper sanitization.

Network Security: Ensure that network traffic between the Oracle database server and the Gemini API endpoint is secured using HTTPS.

Least Privilege: Grant only the necessary privileges to the Oracle user executing the procedure.

Contributing
Contributions are welcome! Feel free to submit pull requests with bug fixes, enhancements, or new features.

License
This project is licensed under the MIT License.


