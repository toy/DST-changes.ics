require 'erb'
require 'icalendar'
require 'pathname'
require 'progress'
require 'tzinfo'
require 'tzinfo/data'

Pathname.class_eval do
  def write!(content)
    dirname.mkpath
    write(content)
  end
end

module Helpers
  def h(text)
    CGI.escapeHTML(text)
  end

  def webcal_link(tz)
    name = tz.name
    css_class = tz.transitions.empty? ? 'blank' : nil
    %Q{<a class="#{css_class}" data-path="#{h name}.ics">#{h name}</a>}
  end
end

class TZ
  def self.all(dates)
    TZInfo::Timezone.all_identifiers.map do |id|
      new(id, dates)
    end
  end

  attr_reader :id, :timezone, :transitions

  def initialize(id, dates)
    @id = id
    @timezone = TZInfo::Timezone.get(id)
    @transitions = @timezone.transitions_up_to(dates.end, dates.begin)
  end

  def name = @timezone.name
end

TZInfo::Timezone.default_dst = false

def init_calendar(name)
  Icalendar::Calendar.new.tap do |c|
    c.append_custom_property('X-WR-CALNAME', name)
    c.append_custom_property('X-PUBLISHED-TTL', 'P1W')
    c.append_custom_property('X-APPLE-CALENDAR-COLOR', '#ffcc33')
  end
end

desc 'build'
task :build do
  include Helpers

  src = Pathname(__dir__)
  dst = Pathname('public')
  dst.rmtree

  today = Date.today
  dates = today.prev_year.to_datetime..today.next_year(2).to_datetime
  all = TZ.all(dates)

  combined_calendar = init_calendar('All DST changes')

  all.with_progress do |tz|
    fail "unsafe name #{tz.name}" unless tz.name =~ %r{\A[a-z0-9_\-+]+(/[a-z0-9_\-+]+)*\z}i

    timezone_calendar = init_calendar("DST changes for #{tz.name}")

    tz.transitions.each do |tt|
      prev = tt.previous_offset
      curr = tt.offset
      diff = curr.observed_utc_offset - prev.observed_utc_offset
      change_description = if diff.zero?
        'no adjustment'
      else
        "clock #{diff.positive? ? 'forward' : 'backward'} by #{format('%d:%02d', *(diff.abs / 60).divmod(60))}"
      end

      timezone_event = timezone_calendar.event
      combined_event = combined_calendar.event

      [timezone_event, combined_event].each do |e|
        e.dtstart = e.dtend = Icalendar::Values::DateTime.new tt.at, 'tzid' => 'UTC'
        e.transp = 'TRANSPARENT'
      end

      timezone_event.summary = "#{prev.abbr} ⇒ #{curr.abbr}\n#{change_description}"
      combined_event.summary = "#{tz.id}:\n#{timezone_event.summary.gsub(/^/, '• ')}"
    end

    dst.join("#{tz.name}.ics").write!(timezone_calendar.to_ical)
  end

  dst.join('combined.ics').write!(combined_calendar.to_ical)

  dst.join('index.html').write!(
    ERB.new(src.join('index.html.erb').read).result(binding)
  )

  %w[favicon.svg robots.txt].each do |basename|
    dst.join(basename).write!(src.join(basename).read)
  end
end
