# matomo_live_user_details
#this script is use to get the data of live visitors  


var API = {
    params: {
        apiUrl: 'https://analytics.cl.telemetrics.tech/index.php?module=API&method=Live.getLastVisitsDetails&idSite={$ID_SITE}&period=day&date=today&format=json&token_auth={$TOKEN_AUTH}'
    },

    fetchData: function () {
        var request = new HttpRequest();

        if (typeof this.params.proxy !== 'undefined' && this.params.proxy !== '') {
            request.setProxy(this.params.proxy);
        }

        var response = request.get(this.params.apiUrl);

        Zabbix.log(4, '[API Request] Response: ' + response);

        if (request.getStatus() !== 200) {
            throw 'Request failed with status code ' + request.getStatus() + ': ' + response;
        }

        try {
            var data = JSON.parse(response);
        } catch (error) {
            throw 'Failed to parse the response. Error: ' + error;
        }

        var currentTimestamp = Math.floor(Date.now() / 1000);
        var lastMinutes = parseInt(API.params.lastMinutes, 10);
        var timeAgo = currentTimestamp - (lastMinutes * 60);

        var filteredData = data.filter(function (visit) {
            if (!visit.firstActionTimestamp || isNaN(visit.firstActionTimestamp)) {
                Zabbix.log(4, '[Skipped Visit] Missing or invalid firstActionTimestamp for visit: ' + JSON.stringify(visit));
                return false;
            }
            return visit.firstActionTimestamp >= timeAgo;
        });

        return filteredData.map(function (visit) {
            return {
                visitIp: visit.visitIp,
                deviceType: visit.deviceType || "Unknown",
                operatingSystem: visit.operatingSystem || "Unknown",
                firstActionTimestamp: visit.firstActionTimestamp || "Unknown",
                browser: visit.browser || "Unknown",
                country: visit.country || "Unknown",
                visitDate: visit.serverDate || "Unknown"
            };
        });
    }
};

try {
    var idSite = '{$ID_SITE}';
    var tokenAuth = '{$TOKEN_AUTH}';
    var lastMinutes = '{$LASTMINUTES}';

    API.params.apiUrl = API.params.apiUrl
        .replace('{$ID_SITE}', idSite)
        .replace('{$TOKEN_AUTH}', tokenAuth);

    API.params.lastMinutes = lastMinutes;

    return JSON.stringify(API.fetchData(), null, 2);
} catch (error) {
    Zabbix.log(3, '[API Request] ERROR: ' + error);
    return JSON.stringify({ 'error': error });
}
