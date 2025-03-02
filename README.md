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
-- ... (rest of the code)
Use code with caution.
Markdown
p_prompt: Input parameter, the text prompt you want to send to the Gemini model.

p_result: Output parameter, a CLOB variable that will store the response from the Gemini model. CLOB is used because responses from LLMs can be very large.

p_debug: Input parameter, a BOOLEAN flag (defaulting to FALSE). If set to TRUE, it will print debug information to the DBMS_OUTPUT stream, aiding in troubleshooting.

p_config_name: Input parameter, the name of the configuration from the llm_config table to use. This allows you to easily switch between different API keys, model names, and endpoints. Defaults to 'default'.

2. Variable Declarations:

l_http_request  UTL_HTTP.req;
  l_http_response UTL_HTTP.resp;
  l_text          VARCHAR2(32767);
  l_api_key       VARCHAR2(100);
  l_url           VARCHAR2(1000);
  l_model_name    VARCHAR2(100);
  l_request_body  CLOB;
  l_status        NUMBER;
  l_reason        VARCHAR2(256);
  l_response_text CLOB;
Use code with caution.
SQL
Variables to handle the HTTP request and response, the API key, the URL, request body, and the response text.

3. Debug Procedure:

PROCEDURE debug_print(p_message IN VARCHAR2) IS
  BEGIN
    IF p_debug THEN
      DBMS_OUTPUT.put_line(p_message);
    END IF;
  END debug_print;
Use code with caution.
SQL
A helper procedure to conditionally print debug messages to the console.

4. Retrieving Configuration:

BEGIN
      SELECT api_key, model_name, api_url
      INTO l_api_key, l_model_name, l_url
      FROM llm_config
      WHERE config_name = p_config_name;
  EXCEPTION
      WHEN NO_DATA_FOUND THEN
          RAISE_APPLICATION_ERROR(-20001, 'Configuration not found: ' || p_config_name);
  END;
Use code with caution.
SQL
Retrieves the API key, model name, and API URL from the llm_config table based on the p_config_name parameter. This allows for storing sensitive information like API keys securely (or at least, more manageable) within the database and switching between different configurations. If no configuration is found, it raises an application error.

5. Creating the Request Body:

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
Use code with caution.
SQL
Constructs the JSON request body that will be sent to the Gemini API. It includes the p_prompt provided as input to the procedure. This is crucial as it formats your request in a way that the Gemini API understands. The structure here directly matches the expected JSON structure of the Google Generative AI API for the generateContent endpoint.

6. Making the HTTP Request:

l_http_request := UTL_HTTP.begin_request(
    url => l_url || '?key=' || l_api_key,
    method => 'POST',
    http_version => 'HTTP/1.1'
  );

  UTL_HTTP.set_header(l_http_request, 'Content-Type', 'application/json');
  UTL_HTTP.set_header(l_http_request, 'Content-Length', LENGTH(l_request_body));

  UTL_HTTP.write_text(l_http_request, l_request_body);

  l_http_response := UTL_HTTP.get_response(l_http_request);
Use code with caution.
SQL
Uses the UTL_HTTP package to create and send an HTTP POST request to the Gemini API. It sets the content type to application/json and the content length. The API key is appended to the URL as a query parameter. Finally, it gets the HTTP response.

Important Security Note: While appending the API key to the URL is common, consider using HTTP headers for a more secure approach in a production environment. This prevents the API key from being logged in server access logs and potentially exposed.

7. Processing the Response:

l_response_text := '';
  BEGIN
    LOOP
      UTL_HTTP.read_text(l_http_response, l_text, 32767);
      l_response_text := l_response_text || l_text;
    END LOOP;
  EXCEPTION
    WHEN UTL_HTTP.end_of_body THEN
      NULL;
  END;

  UTL_HTTP.end_response(l_http_response);
Use code with caution.
SQL
Reads the response from the Gemini API in chunks and appends it to the l_response_text variable. It handles the UTL_HTTP.end_of_body exception to know when the entire response has been read. Finally, it closes the HTTP response.

8. Extracting the Text Content:

IF (l_status = 200) THEN
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
Use code with caution.
SQL
Checks the HTTP status code. If it's 200 (OK), it attempts to parse the JSON response and extract the generated text from the '$.candidates[0].content.parts[0].text' path. This path is specific to the Gemini API response structure. If the JSON parsing fails (perhaps due to an unexpected response format), it simply returns the entire response as the result, along with a debug message. If the status code is not 200, it returns an error message including the reason phrase.

9. Error Handling:

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
Use code with caution.
SQL
A comprehensive EXCEPTION block handles any errors that might occur during the process. It prints the error message and backtrace for debugging and ensures that the HTTP request and response are closed properly, even in case of errors, to prevent resource leaks. The DBMS_UTILITY.FORMAT_ERROR_BACKTRACE is particularly useful for pinpointing the exact line of code where the error occurred. The error handling also includes nested exception blocks to gracefully handle errors while closing the HTTP request/response, preventing cascading failures.

10. Configuration Table (llm_config):

CREATE TABLE llm_config (
    config_name VARCHAR2(50) PRIMARY KEY,
    api_key VARCHAR2(100) NOT NULL,
    model_name VARCHAR2(100) NOT NULL,
    api_url VARCHAR2(500) NOT NULL
);

INSERT INTO llm_config (config_name, api_key, model_name, api_url)
VALUES ('default', 'YOUR_ACTUAL_API_KEY', 'gemini-2.0-flash', 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent');

INSERT INTO llm_config (config_name, api_key, model_name, api_url)
VALUES ('gemini-pro', 'YOUR_ACTUAL_API_KEY', 'gemini-pro', 'https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent');
Use code with caution.
SQL
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
Use code with caution.
SQL
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
Use code with caution.
SQL
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
Use code with caution.
SQL
This example calls the procedure with a prompt and uses the default p_debug value (FALSE). Only the response from the Gemini model will be displayed.

3. Call with a specific Configuration:

SET SERVEROUTPUT ON
DECLARE
    l_result CLOB;
BEGIN
    oracle_call_google_llm('Translate "Hello World" to Spanish.', l_result, FALSE, 'gemini-pro');
    DBMS_OUTPUT.put_line(l_result);
END;
/
Use code with caution.
SQL
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

Key improvements and explanations in this Markdown:

*   **Comprehensive Explanation:**  Each section of the PL/SQL procedure is thoroughly explained, making it easier for users to understand the code's functionality.
*   **Security Emphasis:**  Highlights the importance of API key protection, input validation, and network security.  Advises storing API keys in secure locations and validating user inputs to prevent vulnerabilities.  Suggests using HTTP headers for the API key, rather than the URL.
*   **Detailed Troubleshooting:** Provides a comprehensive list of potential issues and troubleshooting steps, including common errors like `ORA-29273` and permission problems.
*   **`UTL_HTTP` limitations:**  Acknowledges that `UTL_HTTP` may not be the best modern option and suggests `APEX_WEB_SERVICE` as an alternative, when available.
*   **Asynchronous Processing:**  Suggests asynchronous processing for long-running requests.
*   **Customization Options:**  Offers various customization options, such as prompt engineering, error handling enhancements, and rate limiting.
*   **Database Privileges:** Explains the necessity of granting `connect` privilege to the Oracle user using `DBMS_NETWORK_ACL_ADMIN`, providing the exact code and a clear explanation.
*   **JSON Extraction Explained:**  Clearly states that the `JSON_VALUE` extraction path is specific to the Gemini API's response structure.
*   **Concise Style:** Uses bullet points and lists to break up text and make the document easier to read.
*   **Code Formatting:**  Code snippets are formatted for readability.
*   **Practical Examples:** Provides clear and executable usage examples.
*   **Complete Structure:** Includes sections for overview, prerequisites, usage examples, customization, troubleshooting, security considerations, contributing, and license.
*   **Rate Limiting Mentioned:** Added a note about rate limiting to avoid exceeding the API usage limits.
*   **Oracle Wallet Configuration Note:** Mention of the need to configure Oracle Wallet for HTTPS, a common point of confusion.
*   **Clarified API Key Replacement:** Explicitly states to replace 'YOUR_ACTUAL_API_KEY' with a real key.
*   **Error handling is emphasized**, including the `FORMAT_ERROR_BACKTRACE`.
*   **Check for Long responses:** Suggest experimenting with chunk sizes or streaming the response to a file.

This improved Markdown provides a much more comprehensive and user-friendly guide to the PL/SQL procedure. It addresses common issues, emphasizes security best practices, and provides clear instructions for setup and usage.  It is ready to be used as the content for a GitHub repository.  Remember to create the `LICENSE` file according to the MIT license terms!
