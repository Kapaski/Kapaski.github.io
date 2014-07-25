---
layout: post
title: "Logstash to parse json with json arrays in values"
date: 2014-07-24 17:15:06 +1000
comments: true
categories: Logstash
---

Logstash has a known issue that it doesn't convert json array into hash but just return the array.

Example

{a:[11,22,33]} gives you `a = [11,22,33]` << this is correct
{a:[{foo:11}, {foo:22}]} gives you `a = [{foo:11}, {foo:22}] ` << this is not flat enough, especially some queries are requiring to use keys like `a.foo = 11`

By all means, there a couple of pull request to the Logstash github :
https://github.com/elasticsearch/logstash/pull/1185
but neither of them has been merged. There are still below comments in [source code](https://github.com/elasticsearch/logstash/blob/master/lib/logstash/filters/json.rb) today:



      # TODO(sissel): Note, this will not successfully handle json lists
      # like your text is '[ 1,2,3 ]' json parser gives you an array (correctly)
      # which won't merge into a hash. If someone needs this, we can fix it
      # later.



So standing in middle of nowhere, with what's on hand, the `Ruby` filter seems the last resort, and below code works, so no need to use the out of box 'json' filter anymore

      input {
        stdin{}
      }

      filter {
        grok {
          match => ["message","(?<json_raw>.*)"]
        }

        ruby {
          init => "
            def parse_json obj, pname=nil, event

              obj = JSON.parse(obj) unless obj.is_a? Hash
              obj = obj.to_hash unless obj.is_a? Hash

              obj.each {|k,v|
                p = pname.nil?? k : pname
                if v.is_a? Array
                  v.each_with_index {|oo,ii|

                    parse_json_array(oo,ii,p,event)
                  }
                elsif v.is_a? Hash
                  parse_json(v,p,event)
                else

                  event[p] = v
                end
              }

            end

            def parse_json_array obj, i,pname, event

              obj = JSON.parse(obj) unless obj.is_a? Hash
              pname_ = pname
              if obj.is_a? Hash
                obj.each {|k,v|

                  p=[pname_,i,k].join('.')
                  if v.is_a? Array
                    v.each_with_index {|oo,ii|
                      parse_json_array(oo,ii,p,event)
                    }
                  elsif v.is_a? Hash
                    parse_json(v,p, event)
                  else
                    event[p] = v
                  end
                }
              else
                n = [pname_, i].join('.')
                event[n] = obj
              end
            end
          "
          code => "parse_json(event['json_raw'].to_s,nil,event) if event['json_raw'].to_s.include? ':'"
        }


      }

      output {
        stdout{codec => rubydebug}
      }






Test json structure


    {"id":123, "members":[{"i":1, "arr":[{"ii":11},{"ii":22}]},{"i":2}], "im_json":{"id":234, "members":[{"i":3},{"i":4}]}}

and this is whats output

          {
               "message" => "{\"id\":123, \"members\":[{\"i\":1, \"arr\":[{\"ii\":11},{\"ii\":22}]},{\"i\":2}], \"im_json\":{\"id\":234, \"members\":[{\"i\":3},{\"i\":4}]}}",
              "@version" => "1",
            "@timestamp" => "2014-07-25T00:06:00.814Z",
                  "host" => "Leis-MacBook-Pro.local",
              "json_raw" => "{\"id\":123, \"members\":[{\"i\":1, \"arr\":[{\"ii\":11},{\"ii\":22}]},{\"i\":2}], \"im_json\":{\"id\":234, \"members\":[{\"i\":3},{\"i\":4}]}}",
                    "id" => 123,
           "members.0.i" => 1,
    "members.0.arr.0.ii" => 11,
    "members.0.arr.1.ii" => 22,
           "members.1.i" => 2,
               "im_json" => 234,
           "im_json.0.i" => 3,
           "im_json.1.i" => 4
          }
