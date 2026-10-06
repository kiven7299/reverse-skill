# Automation entry

Preferred open sequence:

1. `js-reverse_new_page` or `js-reverse_navigate_page` to open the page
2. `js-reverse_list_network_requests` for recent requests
3. `js-reverse_get_request_initiator` for the stack
4. `js-reverse_list_scripts` to bound scripts
5. `js-reverse_search_in_sources` for request paths, param names, function names
6. If needed, `js-reverse_break_on_xhr` or `js-reverse_set_breakpoint_on_text`

Do not start by guessing how to stub `window`, `document`, `navigator`.
