# Boxr

[![Gem Version](https://badge.fury.io/rb/boxr.svg)](https://badge.fury.io/rb/boxr)

Boxr is a Ruby client library for the Box V2 Content API.  Box employees affectionately refer to one another as Boxers, hence the name of this gem.

The purpose of this gem is to provide a clear, efficient, and intentional method of interacting with the Box Content API. As with any SDK that wraps a REST API, it is important to fully understand the Box Content API at the REST endpoint level.  You are strongly encouraged to read through the Box documentation located [here](https://developer.box.com/en/reference/).

The full RubyDocs for Boxr can be found [here](http://www.rubydoc.info/gems/boxr/Boxr/Client).  You are also encouraged to rely heavily on the source code found in the [lib/boxr](https://github.com/cburnette/boxr/tree/master/lib/boxr) directory of this gem, as well as on the integration tests found [here](https://github.com/cburnette/boxr/blob/master/spec/boxr_spec.rb).

## Versioning

Boxr follows Semantic Versioning since version 1.5.0

## Requirements
This gem requires Ruby 2.0.0 or higher.

## Installation
Add this line to your application's Gemfile:

```ruby
gem 'boxr'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install boxr

If you're going to be using chunked uploads, you will need to add `parallel` if you want to upload multiple chunks concurrently (i.e., `n_threads > 1`). Add to Gemfile:

```ruby
gem 'parallel', '~> 1.0'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install parallel --version '~> 1.0'

## Usage
Super-fast instructions:

1. go to http://developers.box.com
2. find or create your Box Content API app for testing
3. click 'Edit Application'
4. check the boxes for 'Read and write all files and folders' and 'Manage an enterprise'
5. click 'Create a developer token'
6. copy the token and use it in the code below in place of {BOX_DEVELOPER_TOKEN}

```ruby
require 'boxr'

client = Boxr::Client.new('{BOX_DEVELOPER_TOKEN}')
items = client.folder_items(Boxr::ROOT)
items.each {|i| puts i.name}
```

### Creating a client
There are a few different ways to create a Boxr client.  The simplest is to use a Box Developer Token (you generate these from your Box app's General Information page).  They last for 60 minutes and you don't have to go through OAuth2.

```ruby
client = Boxr::Client.new('yPDWOvnumUFaKIMrNBg6PGJpWXC0oaFW')

# Alternatively, you can set an environment variable called BOX_DEVELOPER_TOKEN to
# the value of your current token. By default, Boxr will look for that.

client = Boxr::Client.new  #uses ENV['BOX_DEVELOPER_TOKEN']
```

The next way is to use an access token retrieved after going through the OAuth2 process.  If your application is going to handle refreshing the tokens in a scheduled way (more on this later) then this is the way to go.

```ruby
client = Boxr::Client.new('v2eAXqhZ28WIEpIWeAJcmyamLLt77icP')  #a valid OAuth2 access token

# Boxr will raise an error if this token becomes invalid. It is up to your application to generate
# a new pair of access and refresh tokens in a timely manner.
```

If you want Boxr to automatically refresh the tokens once the access token becomes invalid you can supply a refresh token, along with your client_id and client_secret, and a block that will get invoked when the refresh occurs.

```ruby
token_refresh_callback = lambda {|access, refresh, identifier| some_method_that_saves_them(access, refresh)}
client = Boxr::Client.new('zX3UjFwNerOy5PSWc2WI8aJgMHtAjs8T',
                          refresh_token: 'dvfzfCQoIcRi7r4Yeuar7mZnaghGWexXlX89sBaRy1hS9e5wFroVVOEM6bs0DwPQ',
                          client_id: 'kplh54vfeagt6jmi4kddg4xdswwvrw8y',
                          client_secret: 'sOsm9ZZ8L8svwrn9FsdulLQVwDizKueU',
                          &token_refresh_callback)

# By default Boxr will look for client_id and client_secret in your environment variables as
# BOX_CLIENT_ID and BOX_CLIENT_SECRET, respectively.  You can omit the two optional parameters above
# if those are present.

# You can provide another parameter called as_user. Read about what that means here: https://developer.box.com/reference#as-user-1

# You can provide yet another parameter called identifier. This can be used, for example, to
# hold the id of the user associated with this Boxr client.  When the callback is invoked this value
# will be provided.
```
Here's the complete method signature to initialize an instance of Boxr::Client
```ruby
initialize( access_token=ENV['BOX_DEVELOPER_TOKEN'],
            refresh_token: nil,
            client_id: ENV['BOX_CLIENT_ID'],
            client_secret: ENV['BOX_CLIENT_SECRET'],
            enterprise_id: ENV['BOX_ENTERPRISE_ID'],
            jwt_private_key: ENV['JWT_PRIVATE_KEY'],
            jwt_private_key_password: ENV['JWT_PRIVATE_KEY_PASSWORD'],
            jwt_public_key_id: ENV['JWT_PUBLIC_KEY_ID'],
            identifier: nil,
            as_user: nil,
            &token_refresh_listener)
```

### A quick example
Before diving into detailed documentation, let's take a look at how to accomplish a simple task with Boxr.  This script will find a specific folder given its path, upload a file to that folder, and create an open shared link to that file.

```ruby
require 'boxr'

client = Boxr::Client.new  #using ENV['BOX_DEVELOPER_TOKEN']

folder = client.folder_from_path('/some/directory/structure')
file = client.upload_file('test.txt', folder)
updated_file = client.create_shared_link_for_file(file, access: :open)
puts "Shared Link: #{updated_file.shared_link.url}"
```

### NOTE: Using HTTP mocking libraries for testing
When using HTTP mocking libraries for testing, you may need to set Boxr::BOX_CLIENT to a fresh instance of HTTPClient in your test setup after loading the HTTP mocking library. For example, when using WebMock with RSpec you might could add the following to your RSpec configuration:
``` ruby
RSpec.configure do |config|
  config.before(:suite) do
    Boxr::BOX_CLIENT = HTTPClient.new
  end
end
```

### Methods
#### [OAuth & JWT](https://developer.box.com/en/reference/resources/access-token/)
```ruby
#NOTE: these are all module methods

#OAuth methods
Boxr::oauth_url(state, host: "app.box.com", response_type: "code", scope: nil, folder_id: nil, client_id: ENV['BOX_CLIENT_ID'])

Boxr::get_tokens(code=nil, grant_type: "authorization_code", assertion: nil, scope: nil, username: nil, client_id: ENV['BOX_CLIENT_ID'], client_secret: ENV['BOX_CLIENT_SECRET'])

Boxr::refresh_tokens(refresh_token, client_id: ENV['BOX_CLIENT_ID'], client_secret: ENV['BOX_CLIENT_SECRET'])

Boxr::revoke_tokens(token, client_id: ENV['BOX_CLIENT_ID'], client_secret: ENV['BOX_CLIENT_SECRET'])

#JWT methods
Boxr::get_enterprise_token(private_key: ENV['JWT_PRIVATE_KEY'], private_key_password: ENV['JWT_PRIVATE_KEY_PASSWORD'], public_key_id: ENV['JWT_PUBLIC_KEY_ID'], enterprise_id: ENV['BOX_ENTERPRISE_ID'], client_id: ENV['BOX_CLIENT_ID'], client_secret: ENV['BOX_CLIENT_SECRET'])

Boxr::get_user_token(user_id, private_key: ENV['JWT_PRIVATE_KEY'], private_key_password: ENV['JWT_PRIVATE_KEY_PASSWORD'], public_key_id: ENV['JWT_PUBLIC_KEY_ID'], client_id: ENV['BOX_CLIENT_ID'], client_secret: ENV['BOX_CLIENT_SECRET'])
```
#### [Folders](https://developer.box.com/en/reference/resources/folder/)
```ruby
folder_from_path(path)

folder_from_id(folder_id, fields: [])
alias :folder :folder_from_id

folder_items(folder, fields: [], offset: 0, limit: FOLDER_ITEMS_LIMIT)

root_folder_items(fields: [], offset: 0, limit: FOLDER_ITEMS_LIMIT)

create_folder(name, parent)

update_folder(folder, name: nil, description: nil, parent: nil, shared_link: nil,
                folder_upload_email_access: nil, owned_by: nil, sync_state: nil, tags: nil,
                can_non_owners_invite: nil, if_match: nil)

move_folder(folder, new_parent, name: nil, if_match: nil)

delete_folder(folder, recursive: false, if_match: nil)

copy_folder(folder, dest_folder, name: nil)

create_shared_link_for_folder(folder, access: nil, unshared_at: nil, can_download: nil, can_preview: nil, password: nil)

disable_shared_link_for_folder(folder)

trash(fields: [], offset: 0, limit: FOLDER_ITEMS_LIMIT)

trashed_folder(folder, fields: [])

delete_trashed_folder(folder)

restore_trashed_folder(folder, name: nil, parent: nil)
```
#### [Files](https://developer.box.com/en/reference/resources/file/)
```ruby
file_from_path(path)

file_from_id(file_id, fields: [])
alias :file :file_from_id

def embed_url(file, show_download: false, show_annotations: false)
alias :embed_link :embed_url
alias :preview_url :embed_url
alias :preview_link :embed_url

update_file(file, name: nil, description: nil, parent: nil, shared_link: nil, tags: nil, if_match: nil)

lock_file(file, expires_at: nil, is_download_prevented: false, if_match: nil)

unlock_file(file, if_match: nil)

move_file(file, new_parent, name: nil, if_match: nil)

download_file(file, version: nil, follow_redirect: true)

download_url(file, version: nil)

upload_file(path_to_file, parent, content_created_at: nil, content_modified_at: nil,
            preflight_check: true, send_content_md5: true)

upload_file_from_io(io, parent, name:, content_created_at: nil, content_modified_at: nil,
                     preflight_check: true, send_content_md5: true)

delete_file(file, if_match: nil)

upload_new_version_of_file(path_to_file, file, content_modified_at: nil, send_content_md5: true,
                            preflight_check: true, if_match: nil)

upload_new_version_of_file_from_io(io, file, name: nil, content_modified_at: nil, send_content_md5: true,
                                    preflight_check: true, if_match: nil)

versions_of_file(file)

promote_old_version_of_file(file, file_version)

delete_old_version_of_file(file, file_version, if_match: nil)

copy_file(file, parent, name: nil)

thumbnail(file, min_height: nil, min_width: nil, max_height: nil, max_width: nil)

create_shared_link_for_file(file, access: nil, unshared_at: nil, can_download: nil, can_preview: nil, password: nil)

disable_shared_link_for_file(file)

trashed_file(file, fields: [])

delete_trashed_file(file)

restore_trashed_file(file, name: nil, parent: nil)
```
#### [Chunked Uploads](https://developer.box.com/en/reference/resources/session-endpoints/)
```ruby
chunked_upload_create_session_new_file(path_to_file, parent, name: nil)

chunked_upload_create_session_new_file_from_io(io, parent, name)

chunked_upload_create_session_new_version(path_to_file, file, name: nil)

chunked_upload_create_session_new_version_from_io(io, file, name)

chunked_upload_get_session(session_id)

chunked_upload_part(path_to_file, session_id, content_range)

chunked_upload_part_from_io(io, session_id, content_range)

chunked_upload_list_parts(session_id, limit: nil, offset: nil)

chunked_upload_commit(path_to_file, session_id, parts, content_created_at: nil, content_modified_at: nil, if_match: nil, if_non_match: nil)

chunked_upload_commit_from_io(io, session_id, parts, content_created_at: nil, content_modified_at: nil, if_match: nil, if_non_match: nil)

chunked_upload_abort_session(session_id)

chunked_upload_file(path_to_file, parent, name: nil, n_threads: 1, content_created_at: nil, content_modified_at: nil)

chunked_upload_file_from_io(io, parent, name, n_threads: 1, content_created_at: nil, content_modified_at: nil)

chunked_upload_new_version_of_file(path_to_file, file, name: nil, n_threads: 1, content_created_at: nil, content_modified_at: nil)

chunked_upload_new_version_of_file_from_io(io, file, name, n_threads: 1, content_created_at: nil, content_modified_at: nil)
```
#### [Web Links](https://developer.box.com/en/reference/resources/web-link/)
```ruby
create_web_link(url, parent, name: nil, description: nil)

get_web_link(web_link)

update_web_link(web_link, url: nil, parent: nil, name: nil, description: nil)

delete_web_link(web_link)

trashed_web_link(web_link, fields: [])

delete_trashed_web_link(web_link)

restore_trashed_web_link(web_link, name: nil, parent: nil)
```
#### [Comments](https://developer.box.com/en/reference/resources/comment/)
```ruby
file_comments(file, fields: [], offset: 0, limit: DEFAULT_LIMIT)

add_comment_to_file(file, message: nil, tagged_message: nil)

reply_to_comment(comment, message: nil, tagged_message: nil)

change_comment(comment, message)

comment_from_id(comment_id, fields: [])
alias :comment :comment_from_id

delete_comment(comment)
```
#### [Collaborations](https://developer.box.com/en/reference/resources/collaboration/)
```ruby
folder_collaborations(folder)

add_collaboration(folder, accessible_by, role, fields: [], notify: nil)

edit_collaboration(collaboration, role: nil, status: nil)

remove_collaboration(collaboration)

collaboration_from_id(collaboration_id, fields: [], status: nil)
alias :collaboration :collaboration_from_id

pending_collaborations()
```
#### [Events](https://developer.box.com/en/reference/resources/event/)
```ruby
user_events(stream_position, stream_type: :all, limit: 800)

enterprise_events(created_after: nil, created_before: nil, stream_position: 0, event_type: nil, limit: 500)

enterprise_events_stream(initial_stream_position, event_type: nil, limit: 500, refresh_period: 300)
```
#### [Shared Items](https://developer.box.com/en/reference/get-shared-items/)
```ruby
shared_item(shared_link, shared_link_password: nil)
```
#### [Search](https://developer.box.com/en/reference/get-search/)
```ruby
search( query=nil, scope: nil, file_extensions: [],
        created_at_range_from_date: nil, created_at_range_to_date: nil,
        updated_at_range_from_date: nil, updated_at_range_to_date: nil,
        size_range_lower_bound_bytes: nil, size_range_upper_bound_bytes: nil,
        owner_user_ids: [], ancestor_folder_ids: [], content_types: [], trash_content: nil,
        mdfilters: nil, type: nil, limit: 30, offset: 0)
```
#### [Users](https://developer.box.com/en/reference/resources/user/)
```ruby
current_user(fields: [])
alias :me :current_user

user_from_id(user_id, fields: [])
alias :user :user_from_id

all_users(filter_term: nil, fields: [], offset: 0, limit: DEFAULT_LIMIT)


create_user(name, login: nil, role: nil, language: nil, is_sync_enabled: nil, job_title: nil,
            phone: nil, address: nil, space_amount: nil, tracking_codes: nil,
            can_see_managed_users: nil, is_external_collab_restricted: nil, status: nil, timezone: nil,
            is_exempt_from_device_limits: nil, is_exempt_from_login_verification: nil,
            is_platform_access_only: nil)


update_user(user, notify: nil, enterprise: true, name: nil, role: nil, language: nil, is_sync_enabled: nil,
            job_title: nil, phone: nil, address: nil, space_amount: nil, tracking_codes: nil,
            can_see_managed_users: nil, status: nil, timezone: nil, is_exempt_from_device_limits: nil,
            is_exempt_from_login_verification: nil, is_exempt_from_reset_required: nil, is_external_collab_restricted: nil)

email_aliases_for_user(user)

add_email_alias_for_user(user, email)

remove_email_alias_for_user(user, email_alias)

delete_user(user, notify: nil, force: nil)

move_users_folder(user, source_folder = 0, destination_user)
```
#### [Groups](https://developer.box.com/en/reference/resources/group/)
```ruby
groups(fields: [], offset: 0, limit: DEFAULT_LIMIT)

create_group(name)

update_group(group, name)
alias :rename_group :update_group

delete_group(group)

group_memberships(group, offset: 0, limit: DEFAULT_LIMIT)

group_memberships_for_user(user, offset: 0, limit: DEFAULT_LIMIT)

group_memberships_for_me(offset: 0, limit: DEFAULT_LIMIT)

group_membership_from_id(membership_id)
alias :group_membership :group_membership_from_id

add_user_to_group(user, group, role: nil)

update_group_membership(membership, role)

delete_group_membership(membership)

group_collaborations(group, offset: 0, limit: DEFAULT_LIMIT)
```
#### [Tasks](https://developer.box.com/en/reference/resources/task/)
```ruby
file_tasks(file, fields: [])

create_task(file, action: :review, message: nil, due_at: nil)

task_from_id(task_id)
alias :task :task_from_id

update_task(task, action: :review, message: nil, due_at: nil)

delete_task(task)

task_assignments(task)

create_task_assignment(task, assign_to: nil, assign_to_login: nil)

task_assignment(task)

delete_task_assignment(task)

update_task_assignment(task, message: nil, resolution_state: nil)
```
#### [Metadata](https://developer.box.com/en/reference/resources/metadata/)
```ruby
create_metadata(file, metadata, scope: :global, template: :properties)
create_folder_metadata(folder, metadata, scope, template)

metadata(file, scope: :global, template: :properties)
folder_metadata(folder, scope, template)

all_metadata(file)
all_folder_metadata(folder)

update_metadata(file, updates, scope: :global, template: :properties)
update_folder_metadata(folder, updates, scope, template)

delete_metadata(file, scope: :global, template: :properties)
delete_folder_metadata(folder, scope, template)

get_enterprise_templates
get_metadata_template_by_name(scope, template_key)
get_metadata_template_by_id(template_id)

create_metadata_template(display_name, template_key: nil, fields: [], hidden: nil)
delete_metadata_template(scope, template_key)
```

#### [Watermarking](https://developer.box.com/en/reference/resources/watermark/)
```ruby
get_watermark_on_file(file)

apply_watermark_on_file(file)

remove_watermark_on_file(file)

get_watermark_on_folder(folder)

apply_watermark_on_folder(folder)

remove_watermark_on_folder(folder)
```
## Contributing

1. Fork it ( https://github.com/cburnette/boxr/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
